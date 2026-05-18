# URL Shortener — Kubernetes (Project 2)

> Deploying the [URL Shortener](https://github.com/Priyank-28/url-shortener) (FastAPI + PostgreSQL + Redis) to Kubernetes using Minikube. Built as part of a cloud engineering portfolio targeting top MNC roles.

---

## Architecture

```
Your Browser
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster (Minikube)               │
│                                                             │
│   ┌─────────────────────────────────┐                       │
│   │      Ingress (nginx)            │                       │
│   │  url-shortener.local → url-api  │                       │
│   └────────────────┬────────────────┘                       │
│                    │                                        │
│   ┌────────────────▼────────────────┐   ┌──────────────┐   │
│   │     Service: url-api            │◄──│     HPA      │   │
│   │     ClusterIP · port 80→8000    │   │  min:2 max:5 │   │
│   └──────────┬──────────┬───────────┘   └──────────────┘   │
│              │          │                                   │
│   ┌──────────▼──┐  ┌────▼────────┐                         │
│   │  API Pod 1  │  │  API Pod 2  │  ← Deployment           │
│   │  FastAPI    │  │  FastAPI    │    (manages replicas,    │
│   │  cpu:100m   │  │  cpu:100m   │     rolling updates,     │
│   └──────┬──────┘  └─────┬───────┘     self-healing)       │
│          │               │                                  │
│    ┌─────▼───────┐  ┌────▼──────────┐                      │
│    │ConfigMap    │  │    Secret     │                       │
│    │6 env vars   │  │ 2 credentials │                       │
│    └─────────────┘  └───────────────┘                      │
│                                                             │
│   ┌──────────────────┐   ┌──────────────────┐              │
│   │ Service: postgres│   │  Service: redis  │              │
│   │ ClusterIP · 5432 │   │ ClusterIP · 6379 │              │
│   └────────┬─────────┘   └────────┬─────────┘              │
│            │                      │                        │
│   ┌────────▼─────────┐   ┌────────▼─────────┐              │
│   │   Postgres Pod   │   │    Redis Pod     │              │
│   │   postgres:15    │   │    redis:7       │              │
│   └──────────────────┘   └──────────────────┘              │
└─────────────────────────────────────────────────────────────┘
```

---

## Kubernetes Objects Used

| Object | Name | Purpose |
|--------|------|---------|
| ConfigMap | `api-config` | Non-sensitive env vars — hosts, ports, db name |
| Secret | `api-secret` | Sensitive credentials — DB user and password |
| Deployment | `url-api` | Manages 2 API pod replicas, rolling updates, self-healing |
| Deployment | `postgres` | Single PostgreSQL pod |
| Deployment | `redis` | Single Redis pod |
| Service | `url-api` | ClusterIP — stable internal endpoint for API pods |
| Service | `postgres` | ClusterIP — DNS hostname `postgres` for DB connections |
| Service | `redis` | ClusterIP — DNS hostname `redis` for cache connections |
| Ingress | `url-ingress` | Routes `url-shortener.local` → `url-api` via nginx |
| HPA | `url-api-hpa` | Auto-scales API pods 2→5 based on CPU utilisation |

---

## Project Structure

```
url-shortener-k8s/
└── k8s/
    ├── api/
    │   ├── configmap.yaml      # POSTGRES_HOST, REDIS_HOST, ports, db name
    │   ├── secret.yaml         # POSTGRES_USER, POSTGRES_PASSWORD
    │   ├── deployment.yaml     # 2 replicas, resource requests, probes
    │   ├── service.yaml        # ClusterIP port 80→8000
    │   └── hpa.yaml            # CPU target 50%, min 2 max 5 pods
    ├── postgres/
    │   ├── deployment.yaml     # postgres:15, env from Secret+ConfigMap
    │   └── service.yaml        # ClusterIP port 5432
    ├── redis/
    │   ├── deployment.yaml     # redis:7
    │   └── service.yaml        # ClusterIP port 6379
    └── ingress/
        └── ingress.yaml        # nginx, url-shortener.local
```

---

## Prerequisites

- Docker Desktop (with WSL2 backend)
- WSL2 (Ubuntu)
- `minikube` installed in WSL
- `kubectl` installed in WSL

```bash
# Install kubectl in WSL
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Install minikube in WSL
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube-linux-amd64 && sudo mv minikube-linux-amd64 /usr/local/bin/minikube
```

---

## Setup & Deploy

### 1. Start the cluster

```bash
minikube start --driver=docker --cpus=2 --memory=2048
```

### 2. Point Docker CLI at Minikube's daemon

```bash
eval $(minikube docker-env)
```

> **Important:** Run this in every new terminal session. Without it, `docker build` targets your local daemon — Minikube's kubelet won't find the image.

### 3. Build the app image into Minikube

```bash
cd ~/projects/url-shortner
docker build -t url-shortener:v1 .
```

We use tag `v1` not `latest` — Kubernetes `imagePullPolicy` for `latest` is `Always` (tries Docker Hub, fails). A specific tag defaults to `IfNotPresent` (uses local image).

### 4. Enable addons

```bash
minikube addons enable ingress          # nginx Ingress Controller
minikube addons enable metrics-server   # required for HPA
```

### 5. Deploy all objects

```bash
# Config first — pods depend on these
kubectl apply -f k8s/api/configmap.yaml
kubectl apply -f k8s/api/secret.yaml

# Databases before API — API crashes without them
kubectl apply -f k8s/postgres/deployment.yaml
kubectl apply -f k8s/postgres/service.yaml
kubectl apply -f k8s/redis/deployment.yaml
kubectl apply -f k8s/redis/service.yaml

# API layer
kubectl apply -f k8s/api/deployment.yaml
kubectl apply -f k8s/api/service.yaml
kubectl apply -f k8s/ingress/ingress.yaml
kubectl apply -f k8s/api/hpa.yaml
```

### 6. Verify everything is running

```bash
kubectl get pods
kubectl get services
kubectl get ingress
kubectl get hpa
```

Expected output:

```
NAME                       READY   STATUS    RESTARTS   AGE
postgres-xxx               1/1     Running   0          2m
redis-xxx                  1/1     Running   0          2m
url-api-xxx                1/1     Running   0          2m
url-api-xxx                1/1     Running   0          2m

NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP
postgres     ClusterIP   10.x.x.x         <none>        5432/TCP
redis        ClusterIP   10.x.x.x         <none>        6379/TCP
url-api      ClusterIP   10.x.x.x         <none>        80/TCP

NAME          CLASS   HOSTS                 ADDRESS        PORTS
url-ingress   nginx   url-shortener.local   192.168.49.2   80

NAME          REFERENCE            TARGETS      MINPODS   MAXPODS   REPLICAS
url-api-hpa   Deployment/url-api   cpu: 4%/50%  2         5         2
```

---

## Testing

### Port-forward (recommended for WSL2)

```bash
kubectl port-forward service/url-api 8080:80
```

In a second terminal:

```bash
# Health check
curl http://localhost:8080/health
# {"status":"ok"}

# Shorten a URL
curl -X POST http://localhost:8080/shorten \
  -H "Content-Type: application/json" \
  -d '{"url": "https://github.com/Priyank-28"}'
# {"short_code":"KUehEr","short_url":"...","original_url":"..."}

# Redirect
curl -L http://localhost:8080/KUehEr
```

### Internal cluster test

```bash
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- \
  curl -s http://url-api/health
```

---

## Key Concepts Explained

### Desired state reconciliation
You declare what you want (`replicas: 2`). Kubernetes watches the cluster forever and ensures reality matches the declaration. Pod crashes → controller creates a replacement automatically.

### How Services and DNS work
Every Service gets a stable ClusterIP and a DNS name equal to `metadata.name`. When the API container connects to hostname `postgres`, Kubernetes DNS resolves it to the postgres Service IP, which load-balances to the matching pod. The only config change from Docker Compose was renaming `POSTGRES_HOST=db` to `POSTGRES_HOST=postgres`.

### ConfigMap vs Secret
Both inject environment variables into pods. ConfigMap holds non-sensitive config (hostnames, ports). Secret holds credentials — values are base64-encoded at rest and never printed by `kubectl describe`. In production, Secrets are backed by HashiCorp Vault or AWS Secrets Manager.

### Readiness vs Liveness probes
- **Readiness** — is this pod ready to receive traffic? Until `/health` returns 200, the pod is excluded from the Service's load balancer.
- **Liveness** — is this pod still healthy? If `/health` fails, Kubernetes kills and restarts the pod.

### Resource requests vs limits
- **requests** — minimum guaranteed CPU/memory. Used by the scheduler to place pods and by HPA to calculate utilisation percentage.
- **limits** — maximum allowed. Exceed CPU limit → throttled. Exceed memory limit → OOMKilled and restarted.

### HPA calculation
```
current replicas × (actual CPU / requested CPU) = desired replicas
```
At `cpu: 4%/50%` with 2 pods → no scale needed. At `cpu: 80%/50%` → HPA adds pods until average drops below 50%.

---

## Stopping and Resuming

```bash
# Stop (preserves all state)
minikube stop

# Resume
minikube start --driver=docker --cpus=2 --memory=2048
eval $(minikube docker-env)
```

---

## Lessons Learned

- **WSL2 + Minikube**: Install both `kubectl` and `minikube` natively inside WSL2 — avoid the Windows installations entirely. The Windows/WSL path translation (`C:\Users\...` vs `/mnt/c/...`) causes kubeconfig cert errors that are painful to debug.
- **`eval $(minikube docker-env)`**: Must be run per terminal session. Forget this and your image builds go to the wrong daemon — kubelet can't find the image and pods stay in `ImagePullBackOff`.
- **Deploy order matters**: ConfigMap and Secret must exist before Deployments that reference them. Databases must be running before the API, otherwise the API crashes on startup (Kubernetes restarts it automatically, but it's cleaner to deploy in order).
- **Readiness probes are your friend**: The API pods showed `0/1 Running` while postgres wasn't ready — the probe correctly blocked traffic until the app was actually healthy.

---

## Interview Cheat Sheet

**Q: What is Kubernetes and why use it over Docker Compose?**
Compose runs on one machine — if it dies, everything dies, and you can't scale. Kubernetes treats a fleet of machines as a single pool, enforces desired state forever, and scales automatically.

**Q: What is the difference between a Deployment and a Pod?**
A Pod is a single container instance. A Deployment manages a set of identical Pods — maintains replica count, handles rolling updates, restarts crashed pods. You never create Pods directly in production.

**Q: How does Kubernetes DNS work?**
Every Service gets a DNS name equal to its `metadata.name`. The API connects to hostname `postgres` — Kubernetes DNS resolves it to the postgres Service ClusterIP, which forwards to the pod.

**Q: Why split config into ConfigMap and Secret?**
ConfigMap is for non-sensitive config that you'd commit to Git. Secret is for credentials that must never be committed. In production, Secrets are backed by Vault or AWS Secrets Manager, not stored in etcd directly.

**Q: How does HPA work?**
It polls Metrics Server every 15 seconds. Calculates `actual CPU / requested CPU`. If average across all pods exceeds the threshold, it increases `replicas` on the Deployment. Controller creates new pods automatically.

---

## Related Projects

- **Project 1** — [url-shortener](https://github.com/Priyank-28/url-shortener) — FastAPI + PostgreSQL + Redis in Docker Compose
- **Project 3** — Coming soon: Terraform on AWS
