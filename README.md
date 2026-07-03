# ☸️ Kubernetes Production-Style Deployment

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?style=flat&logo=helm&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat&logo=grafana&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?style=flat&logo=nginx&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green.svg)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)

> **Project 3 of 3** in my DevOps portfolio series.
> [Project 1 — CI/CD Pipeline](https://github.com/Hacker3S/cicd-docker-pipeline) · [Project 2 — Dockerized Multi-Service App](https://github.com/Hacker3S/dockerized-multi-service-app)

Taking the multi-service app from Project 2 and deploying it on **Kubernetes** with an Ingress controller, load-balanced replicas, persistent storage, and full cluster monitoring via Prometheus + Grafana — the way it would run in a real production environment.

---

## 📐 Architecture

```
                         Internet / Browser
                                │
                                ▼
                    ┌────────────────────────┐
                    │   Load Balancer          │
                    │   (Ingress Controller    │
                    │    Service)              │
                    └────────────────────────┘
                                │
                                ▼
                    ┌────────────────────────┐
                    │   Ingress (nginx)        │  routes by URL path
                    └────────────────────────┘
                       │                  │
                "/"    │                  │  "/api"
                       ▼                  ▼
            ┌──────────────┐      ┌──────────────┐
            │  Frontend     │      │  Backend      │
            │  Deployment   │      │  Deployment   │
            │  (2 replicas) │      │  (2 replicas) │
            └──────────────┘      └──────┬───────┘
                                          │
                                          ▼
                                  ┌──────────────┐
                                  │  PostgreSQL   │
                                  │  Deployment   │
                                  │  + PVC        │
                                  └──────────────┘

     Cluster-wide:  Prometheus (metrics scraping) + Grafana (dashboards)
```

**Key design decisions:**
- The browser only ever talks to one entry point — the Ingress controller
- Nginx Ingress routes `/api/*` to the backend, `/` to the frontend — same pattern as Project 2's reverse proxy, now managed by Kubernetes
- All services communicate internally by **service name**, never by IP — Kubernetes DNS resolves them automatically
- PostgreSQL data is stored in a **PersistentVolumeClaim** so it survives pod restarts and redeployments

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| Orchestration | Kubernetes (Docker Desktop — kubeadm) |
| Ingress / Routing | ingress-nginx |
| Frontend | React 18 (Vite), served via Nginx |
| Backend | Node.js + Express REST API |
| Database | PostgreSQL 16 |
| Monitoring | Prometheus + Grafana (kube-prometheus-stack via Helm) |
| Image Registry | Docker Hub |
| Manifest format | Raw Kubernetes YAML |

---

## 📁 Project Structure

```
kubernetes-production-deployment/
├── README.md
├── .gitignore
└── k8s/
    ├── 00-namespace.yaml              # Isolates all app resources
    ├── 01-postgres-secret.yaml        # DB credentials (Secret)
    ├── 02-postgres-init-configmap.yaml # DB init SQL (ConfigMap)
    ├── 03-postgres-pvc.yaml           # Persistent storage claim
    ├── 04-postgres-deployment.yaml    # PostgreSQL pod
    ├── 05-postgres-service.yaml       # Internal ClusterIP service
    ├── 06-backend-deployment.yaml     # Node.js API — 2 replicas
    ├── 07-backend-service.yaml        # Internal ClusterIP service
    ├── 08-frontend-deployment.yaml    # React app — 2 replicas
    ├── 09-frontend-service.yaml       # Internal ClusterIP service
    └── 10-ingress.yaml                # Path-based routing rules
```

---

## ✅ Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) with **Kubernetes enabled** (Settings → Kubernetes → Enable Kubernetes)
- [kubectl](https://kubernetes.io/docs/tasks/tools/) — Kubernetes CLI
- [Helm](https://helm.sh/docs/intro/install/) — package manager for Kubernetes
- Docker Hub account (images from [Project 2](https://github.com/Hacker3S/dockerized-multi-service-app) pushed there)

---

## 🚀 Deployment Guide

### 1. Confirm the cluster is ready

```powershell
kubectl get nodes
```

Expected output: one node `docker-desktop` with `STATUS: Ready`.

### 2. Install the Ingress controller

Docker Desktop's Kubernetes doesn't bundle an ingress controller, so install ingress-nginx directly:

```powershell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.14.3/deploy/static/provider/cloud/deploy.yaml
```

Wait until the controller pod is running:

```powershell
kubectl get pods -n ingress-nginx
```

### 3. Push the application images to Docker Hub

The Kubernetes manifests pull images from Docker Hub. Build and push from the Project 2 directory:

```powershell
cd ..\dockerized-multi-service-app

docker build -t hacker3s/multi-service-backend:latest ./backend
docker build -t hacker3s/multi-service-frontend:latest ./frontend

docker push hacker3s/multi-service-backend:latest
docker push hacker3s/multi-service-frontend:latest
```

### 4. Deploy all manifests

```powershell
cd ..\kubernetes-production-deployment
kubectl apply -f k8s/
```

`kubectl apply -f` on a directory applies every YAML file inside it in order.

### 5. Watch everything come up

```powershell
kubectl get all -n multiservice
```

Wait until all pods show `Running` and `1/1` or `2/2` ready. This typically takes under 60 seconds since images are already on Docker Hub.

### 6. Add the hostname to your hosts file

Open Notepad as **Administrator** → File → Open → `C:\Windows\System32\drivers\etc\hosts`

Add this line at the bottom:
```
127.0.0.1  multiservice.local
```

Save and close.

### 7. Access the app

```
http://multiservice.local
```

You should see the React app from Project 2 — now served from a Kubernetes cluster with load-balanced replicas behind an Ingress controller.

**Fallback (port-forward) if the hostname doesn't resolve:**
```powershell
kubectl port-forward svc/frontend 8080:80 -n multiservice
```
Then visit `http://localhost:8080`.

---

## 📊 Monitoring — Prometheus + Grafana

### Install the monitoring stack via Helm

```powershell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

Wait for all pods to come up:

```powershell
kubectl get pods -n monitoring
```

> **Note on Windows/WSL2:** The `node-exporter` pod will CrashLoopBackOff — this is a known limitation of running Kubernetes on Docker Desktop for Windows. It tries to read Linux node filesystem paths that don't exist the same way under WSL2. Everything else (Prometheus, Grafana, Alertmanager, kube-state-metrics) runs correctly and all pod-level metrics still work.

### Access Grafana

```powershell
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

Get the admin password:

```powershell
$b64 = kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}"
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($b64))
```

Visit `http://localhost:3000` — login with username `admin` and the password above.

### Useful dashboards (pre-installed)

Navigate to **Dashboards → Browse** in the Grafana sidebar:

- **Kubernetes / Compute Resources / Namespace (Pods)** → select `multiservice` → real-time CPU and memory for every pod
- **Kubernetes / Compute Resources / Cluster** → overall cluster health
- **Kubernetes / Networking / Namespace (Pods)** → network traffic per pod

---

## 🧪 Debugging & Verification Commands

```powershell
# See everything in your namespace at once
kubectl get all -n multiservice

# Logs for a specific deployment (shows logs from one of its pods)
kubectl logs deployment/backend -n multiservice

# Follow logs in real time
kubectl logs -f deployment/backend -n multiservice

# Exec into a running pod
kubectl exec -it deployment/postgres -n multiservice -- psql -U postgres -d appdb -c "SELECT * FROM items;"

# Describe a pod that's stuck (Pending, CrashLoopBackOff, etc.)
kubectl describe pod <pod-name> -n multiservice

# Force a rolling restart of a deployment (no YAML changes needed)
kubectl rollout restart deployment/backend -n multiservice

# Check ingress routing rules
kubectl get ingress -n multiservice

# Delete and redeploy everything cleanly
kubectl delete namespace multiservice
kubectl apply -f k8s/
```

---

## 💡 What This Project Demonstrates

| Concept | Implementation |
|---|---|
| Raw Kubernetes YAML | 11 manifests — Namespace, Secret, ConfigMap, PVC, Deployment, Service, Ingress |
| Secret management | One Secret reused by both Postgres and Backend via `secretKeyRef` under different env var names |
| Ingress routing | Path-based routing (`/api` → backend, `/` → frontend) via ingress-nginx |
| Load balancing | 2 replicas each for frontend and backend; Kubernetes load-balances automatically |
| Persistent storage | PVC ensures PostgreSQL data survives pod restarts and redeployments |
| Health probes | Readiness probes prevent traffic reaching pods that aren't ready; liveness probes restart stuck pods |
| Helm | Entire monitoring stack (Prometheus + Grafana + Alertmanager) deployed with one command |
| Observability | Real-time CPU/memory dashboards per pod in Grafana |

---

## 🔗 Related Projects

| Project | Description | Repo |
|---|---|---|
| Project 1 | CI/CD Pipeline (GitHub Actions + Docker) | [cicd-docker-pipeline](https://github.com/Hacker3S/cicd-docker-pipeline) |
| Project 2 | Dockerized Multi-Service App (Docker Compose) | [dockerized-multi-service-app](https://github.com/Hacker3S/dockerized-multi-service-app) |
| Project 3 | **This project** — Kubernetes Deployment + Monitoring | — |

---

## 🚀 Possible Extensions

- Add a `HorizontalPodAutoscaler` to scale the backend automatically under CPU load
- Add TLS with `cert-manager` + Let's Encrypt for HTTPS on the Ingress
- Package the entire app as a **Helm chart** instead of raw manifests
- Add a custom `/metrics` endpoint to the Node.js backend using `prom-client` and a `ServiceMonitor` so Prometheus scrapes app-level metrics (request count, latency, error rate) — not just cluster metrics
- Deploy to a real cloud Kubernetes cluster (AWS EKS, GCP GKE, or Azure AKS) — the manifests need zero changes

---

## 👤 Author

**Shawn Sreeju Sampath**
[GitHub](https://github.com/Hacker3S) · [LinkedIn](https://linkedin.com/in/shawn-sreeju-sampath-074923377) · [Docker Hub](https://hub.docker.com/u/hacker3s)

---

## 📄 License

This project is licensed under the MIT License.
