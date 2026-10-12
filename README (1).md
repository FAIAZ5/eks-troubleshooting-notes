# EKS troubleshooting notes

In Feb-Mar 2026 I moved my 3-tier app (React frontend, Node/Express backend, PostgreSQL) from Docker Compose onto Amazon EKS. The frontend and backend ran on EKS, the database on RDS, and the images lived in ECR. I did all of it from WSL2 on a Windows laptop.

I kept a log of everything that broke along the way, and these are the five problems worth writing up. The cluster has since been deleted, so none of the endpoints or IPs below are live.

The Docker version of the app is in [3-tier-app](https://github.com/FAIAZ5/3-tier-app).

1. [AccessDenied right after creating the cluster](#1-accessdenied-right-after-creating-the-cluster)
2. [Frontend pods stuck in InvalidImageName](#2-frontend-pods-stuck-in-invalidimagename)
3. [kubectl failing on and off because of WSL2 DNS](#3-kubectl-failing-on-and-off-because-of-wsl2-dns)
4. [Grafana could not reach Prometheus](#4-grafana-could-not-reach-prometheus)
5. [Frontend loads but no data comes through](#5-frontend-loads-but-no-data-comes-through)

## 1. AccessDenied right after creating the cluster

`aws eks create-cluster` went through fine. The next step was waiting for the cluster to become active, and that failed:

```
$ aws eks wait cluster-active --name <cluster>
Waiter ClusterActive failed: An error occurred (AccessDeniedException) when calling the DescribeCluster operation:
User arn:aws:iam::<ACCOUNT_ID>:user/<USER> is not authorized to perform eks:DescribeCluster
```

The cluster itself was fine. My IAM user was allowed to create EKS clusters but not to describe them, and the waiter calls DescribeCluster under the hood.

I wrote a policy for EKS, attached it to my user, then checked it with a read-only call:

```bash
aws iam create-policy --policy-name EKSUserFullAccess --policy-document file://eks-user-policy.json

CURRENT_USER=$(aws sts get-caller-identity --query Arn --output text | cut -d'/' -f2)
aws iam attach-user-policy --user-name $CURRENT_USER \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/EKSUserFullAccess

aws iam list-attached-user-policies --user-name $CURRENT_USER
sleep 30   # IAM changes aren't instant
aws eks list-clusters
```

The policy was `eks:*` on `*`. That got me unblocked, but it's wider than it needs to be. If I did it again I'd only grant the EKS actions the setup actually uses.

Next time: check IAM permissions before starting a long-running create, not halfway through.

## 2. Frontend pods stuck in InvalidImageName

The frontend Deployment applied without errors, but both pods sat in `InvalidImageName`:

```
$ kubectl get pods -n 3tier-app
NAME                        READY   STATUS             RESTARTS   AGE
frontend-7b7c4b5668-5gp8v   0/1     InvalidImageName   0          20h
frontend-7b7c4b5668-7smzr   0/1     InvalidImageName   0          20h
```

The images were definitely in ECR, so I checked what the manifest actually contained:

```
$ grep image kubernetes/frontend-deployment.yaml
image: /3tier-appfrontend:v1.0
```

The registry part of the image name was gone. I'd generated the manifest from a heredoc with `image: $REGISTRY/3tier-appfrontend:v1.0`, but I was in a fresh WSL terminal and `REGISTRY` was never set there. `echo $REGISTRY` printed nothing. Bash expands an unset variable to nothing without any warning, so the mistake only showed up when Kubernetes tried to pull the image.

```bash
export REGISTRY=$(aws sts get-caller-identity --query Account --output text).dkr.ecr.ap-southeast-2.amazonaws.com
echo $REGISTRY

kubectl delete deployment frontend -n 3tier-app
# regenerated the manifest, then checked it before applying
grep image kubernetes/frontend-deployment.yaml
kubectl apply -f kubernetes/frontend-deployment.yaml
```

Both pods were running within a minute. I also added the export to `~/.bashrc` so new terminals pick it up.

Next time: if a manifest is generated from shell variables, grep the image line before applying it.

## 3. kubectl failing on and off because of WSL2 DNS

While I was setting up the backend Service, kubectl started failing at random. `kubectl cluster-info` would work, and then `kubectl get nodes` a few seconds later would fail with:

```
Unable to connect to the server: dial tcp: lookup EA75F3CF7810F023E165E2B15F3A99EA.yl4.ap-southeast-2.eks.amazonaws.com on 10.255.255.254:53: no such host
```

The `10.255.255.254:53` part is what gave it away. That's the DNS server WSL2 configures automatically, not anything in AWS. Running `nslookup` against it resolved the EKS endpoint some of the time, and the rest of the time it returned:

```
;; Got recursion not available from 10.255.255.254
```

So the cluster was most likely fine and my laptop just couldn't reliably resolve its address. I pointed WSL at public resolvers in `/etc/resolv.conf`:

```
nameserver 1.1.1.1
nameserver 8.8.8.8
# nameserver 10.255.255.254
```

After that, `nslookup` and kubectl worked every time, and the nodes had been healthy all along:

```
$ kubectl get nodes
NAME                                               STATUS   ROLES    AGE   VERSION
ip-172-31-14-56.ap-southeast-2.compute.internal    Ready    <none>   7d    v1.29.15-eks-ecaa3a6
ip-172-31-39-143.ap-southeast-2.compute.internal   Ready    <none>   7d    v1.29.15-eks-ecaa3a6
```

WSL rewrites `resolv.conf` on restart, so this edit doesn't survive. To make it stick you set `generateResolvConf = false` in `/etc/wsl.conf`. For this project I just left it as a manual edit.

Next time: when a kubectl error is a DNS lookup failure, check whether the name resolves from my own machine before touching the cluster.

## 4. Grafana could not reach Prometheus

Prometheus and Grafana were both running in the `monitoring` namespace, but adding Prometheus as a data source in Grafana failed:

```
Post "http://prometheus:9090/api/v1/query": dial tcp: lookup prometheus on 10.100.0.10:53: no such host
```

It's the same "no such host" error as case 3, but the DNS server is different. 10.100.0.10 is the cluster's own kube-dns, so this time the name really didn't exist inside the cluster. The Prometheus pod itself was fine:

```
$ kubectl get pods -n monitoring -l app=prometheus
NAME                          READY   STATUS    RESTARTS   AGE
prometheus-7d566dff64-h7zfx   1/1     Running   0          15m

$ kubectl get svc -n monitoring
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)        AGE
grafana   LoadBalancer   10.100.26.167   a4dafdfe30ab6413ca6a77ceb166b3c7-1886803209.ap-southeast-2.elb.amazonaws.com   80:31086/TCP   10m
```

I had a Service for Grafana but had never created one for Prometheus. The Deployment runs the pod, but it's the Service that gives it a DNS name. I added one:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  selector:
    app: prometheus
  ports:
  - port: 9090
    targetPort: 9090
```

After `kubectl apply`, the Service came up as ClusterIP `10.100.143.52`, and Grafana's "Save & test" passed.

Next time: a "no such host" error inside the cluster usually means the Service is missing, or its selector doesn't match the pod labels. Start with `kubectl get svc` and `kubectl get pods --show-labels`.

## 5. Frontend loads but no data comes through

By this point the frontend, backend and RDS were all deployed. The page loaded but showed "Failed to fetch items", and the browser console had:

```
GET http://backend:5000/items net::ERR_NAME_NOT_RESOLVED
```

The backend wasn't the problem. Curling its load balancer directly returned JSON, and the backend logs showed no requests coming in from the frontend at all.

React code runs in the user's browser, not inside the cluster. `backend` is a Kubernetes Service name that only resolves inside the cluster, so my browser had no idea what it was.

The URL came from the build. React bakes `REACT_APP_*` variables into the JavaScript during `npm run build`, and my `.env.production` had:

```
REACT_APP_API_BASE=http://backend:5000
```

I confirmed it had ended up in the built bundle:

```
$ grep -n "API_BASE" build/static/js/main.61b171e5.js.map
"const API_BASE = \"http://backend:5000\""
```

Fixing it took longer than it should have, because I had two copies of the frontend on my machine (an older one in OneDrive and the EKS one), and my first rebuild came from the wrong folder. Once I was in the right one, I:

1. removed the override from `.env.production`
2. pointed the fallback in `App.js` at the backend load balancer's URL
3. rebuilt with `--no-cache` so Docker couldn't reuse the old build layer
4. pushed the new image to ECR and rolled it out

```bash
docker build --no-cache -t <ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com/frontend:v18.0 .
docker push <ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com/frontend:v18.0
kubectl set image deployment/frontend -n 3tier-app frontend=<ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com/frontend:v18.0
kubectl rollout status deployment/frontend -n 3tier-app
```

After a hard refresh (Ctrl+Shift+R), the items loaded and `GET /items 200` started showing up in the backend logs.

Giving the backend its own load balancer worked, but it's not how I'd do it now. When I later rebuilt the app on local Kubernetes, I put nginx in front and proxied `/api` to the backend, so the browser only ever talks to one address.

Next time: for anything the browser calls, ask whether that URL makes sense from the user's laptop.
