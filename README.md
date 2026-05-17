# URL Shortener — Kubernetes

Deploying the [URL Shortener](https://github.com/Priyank-28/url-shortener) to Kubernetes using Minikube.

## Stack
- **App**: FastAPI + PostgreSQL + Redis (from Project 1)
- **Orchestration**: Kubernetes (Minikube locally)
- **Objects**: Deployment, Service, ConfigMap, Secret, Ingress, HPA

## Setup
```bash
minikube start --driver=docker --cpus=2 --memory=2048
kubectl get nodes   # should show minikube Ready
```

## Project Status
- [x] Minikube cluster running
- [ ] API Deployment + Service
- [ ] ConfigMap + Secret
- [ ] PostgreSQL + Redis
- [ ] Ingress
- [ ] HorizontalPodAutoscaler
