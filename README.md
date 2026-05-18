# URL Shortener — Kubernetes

> Deploying the [URL Shortener](https://github.com/Priyank-28/url-shortener) (FastAPI + PostgreSQL + Redis) to Kubernetes using Minikube.

---

## Architecture

```
Your Browser
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster (Minikube)              │
│                                                             │
│   ┌─────────────────────────────────┐                       │
│   │      Ingress (nginx)            │                       │
│   │  url-shortener.local → url-api  │                       │
│   └────────────────┬────────────────┘                       │
│                    │                                        │
│   ┌────────────────▼────────────────┐   ┌──────────────┐    │
│   │     Service: url-api            │◄──│     HPA      │    │
│   │     ClusterIP · port 80→8000    │   │  min:2 max:5 │    │
│   └──────────┬──────────┬───────────┘   └──────────────┘    │
│              │          │                                   │
│   ┌──────────▼──┐  ┌────▼────────┐                          │
│   │  API Pod 1  │  │  API Pod 2  │  ← Deployment            │
│   │  FastAPI    │  │  FastAPI    │    rolling updates +     │
│   │  cpu:100m   │  │  cpu:100m   │    self-healing          │
│   └──────┬──────┘  └─────┬───────┘                          │
│          └──────┬─────────┘                                 │
│    ┌────────────▼──────────────┐                            │
│    │  ConfigMap  │  Secret     │                            │
│    │  6 env vars │  2 creds    │                            │
│    └─────────────┴─────────────┘                            │
│                                                             │
│   ┌──────────────────┐   ┌──────────────────┐               │
│   │ Service: postgres│   │  Service: redis  │               │
│   │ ClusterIP · 5432 │   │ ClusterIP · 6379 │               │
│   └────────┬─────────┘   └────────┬─────────┘               │
│            │                      │                         │
│   ┌────────▼─────────┐   ┌────────▼─────────┐               │
│   │   Postgres Pod   │   │    Redis Pod     │               │
│   │   postgres:15    │   │    redis:7       │               │
│   └──────────────────┘   └──────────────────┘               │
└─────────────────────────────────────────────────────────────┘
```

---

## Kubernetes Objects

| Object | Name | Purpose |
|--------|------|---------|
| ConfigMap | `api-config` | Non-sensitive env vars — hosts, ports, db name |
| Secret | `api-secret` | Credentials — DB user and password |
| Deployment | `url-api` | 2 API pod replicas, rolling updates, self-healing |
| Deployment | `postgres` | Single PostgreSQL pod |
| Deployment | `redis` | Single Redis pod |
| Service | `url-api` | ClusterIP — stable internal endpoint for API pods |
| Service | `postgres` | ClusterIP — DNS hostname `postgres` inside cluster |
| Service | `redis` | ClusterIP — DNS hostname `redis` inside cluster |
| Ingress | `url-ingress` | Routes `url-shortener.local` → `url-api` via nginx |
| HPA | `url-api-hpa` | Auto-scales API pods 2→5 at 50% CPU utilisation |

---

## Project Structure

```
url-shortener-k8s/
└── k8s/
    ├── api/
    │   ├── configmap.yaml      # POSTGRES_HOST, REDIS_HOST, ports, db name
    │   ├── secret.yaml         # POSTGRES_USER, POSTGRES_PASSWORD
    │   ├── deployment.yaml     # 2 replicas, resource requests, health probes
    │   ├── service.yaml        # ClusterIP port 80→8000
    │   └── hpa.yaml            # CPU target 50%, min 2 max 5 pods
    ├── postgres/
    │   ├── deployment.yaml     # postgres:15, env from Secret + ConfigMap
    │   └── service.yaml        # ClusterIP port 5432
    ├── redis/
    │   ├── deployment.yaml     # redis:7
    │   └── service.yaml        # ClusterIP port 6379
    └── ingress/
        └── ingress.yaml        # nginx, host: url-shortener.local
```

---

## Prerequisites

- Docker Desktop with WSL2 backend
- WSL2 (Ubuntu)
- `kubectl` and `minikube` installed natively in WSL

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube-linux-amd64 && sudo mv minikube-linux-amd64 /usr/local/bin/minikube
```

> Install both tools natively inside WSL — not on Windows. The Windows/WSL path boundary causes kubeconfig certificate errors.

---

## Setup

### 1. Start the cluster

```bash
minikube start --driver=docker --cpus=2 --memory=2048
```

### 2. Point Docker CLI at Minikube's daemon

```bash
eval $(minikube docker-env)
```

Run this in every new terminal session. Without it `docker build` targets your local daemon and kubelet cannot find the image.

### 3. Build the app image

```bash
cd ~/projects/url-shortner
docker build -t url-shortener:v1 .
```

Tag `v1` is intentional — `latest` triggers `imagePullPolicy: Always` which attempts a Docker Hub pull and fails for local images. A explicit version tag defaults to `IfNotPresent`.

### 4. Enable addons

```bash
minikube addons enable ingress         # nginx Ingress Controller
minikube addons enable metrics-server  # required for HPA
```

### 5. Deploy

```bash
# Config before pods
kubectl apply -f k8s/api/configmap.yaml
kubectl apply -f k8s/api/secret.yaml

# Databases before API
kubectl apply -f k8s/postgres/deployment.yaml
kubectl apply -f k8s/postgres/service.yaml
kubectl apply -f k8s/redis/deployment.yaml
kubectl apply -f k8s/redis/service.yaml

# API
kubectl apply -f k8s/api/deployment.yaml
kubectl apply -f k8s/api/service.yaml
kubectl apply -f k8s/ingress/ingress.yaml
kubectl apply -f k8s/api/hpa.yaml
```

### 6. Verify

```bash
kubectl get pods
kubectl get services
kubectl get ingress
kubectl get hpa
```

All pods should show `1/1 Running`. HPA target shows `cpu: 4%/50%` once metrics-server has scraped the first sample (~60 seconds).

---

## Testing

Port-forward is the standard method for local testing on WSL2 — Minikube's bridge network (`192.168.49.x`) is not directly routable from WSL2:

```bash
kubectl port-forward service/url-api 8080:80
```

```bash
# Health check
curl http://localhost:8080/health

# Shorten a URL
curl -X POST http://localhost:8080/shorten \
  -H "Content-Type: application/json" \
  -d '{"url": "https://github.com"}'

# Follow redirect
curl -L http://localhost:8080/<short_code>
```

Internal cluster test (no port-forward needed):

```bash
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- \
  curl -s http://url-api/health
```

---

## Design Notes

**Service DNS**: Every Service name becomes a DNS hostname inside the cluster. The API connects to `postgres:5432` and `redis:6379` — Kubernetes CoreDNS resolves these to the respective Service ClusterIPs. The only config change from Docker Compose was `POSTGRES_HOST=db` → `POSTGRES_HOST=postgres` and `REDIS_HOST=cache` → `REDIS_HOST=redis`.

**Health probes**: Readiness probe on `/health` prevents traffic reaching a pod before it has a live database connection. Liveness probe restarts the pod if it becomes unresponsive after startup.

**Resource requests**: `cpu: 100m` on each API container gives HPA a baseline to calculate utilisation against. Without requests, HPA cannot compute a percentage and reports `<unknown>`.

**Deployment order**: ConfigMap and Secret must exist before any Deployment that references them via `envFrom`. Postgres and Redis must be running before the API — the app calls `Base.metadata.create_all()` on startup and crashes if the database is unreachable. Kubernetes restarts crashed pods automatically, but deploying in dependency order avoids unnecessary crash loops.

---

## Stopping and Resuming

```bash
# Stop — preserves all state
minikube stop

# Resume
minikube start --driver=docker --cpus=2 --memory=2048
eval $(minikube docker-env)
```

---

## Related

- **Project 1** — [url-shortener](https://github.com/Priyank-28/url-shortener) — FastAPI + PostgreSQL + Redis on Docker Compose
- **Project 3** — Coming soon: Terraform on AWS
