# ☸️ Kubernetes Production-Style Deployment

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat&logo=grafana&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green.svg)

Taking the multi-service app from [Project 2](https://github.com/Hacker3S/dockerized-multi-service-app) and deploying it on **Kubernetes** with an Ingress controller, a Load Balancer layer, and full cluster monitoring via Prometheus + Grafana.

This is the natural next step after Docker Compose: instead of one machine running four containers, Kubernetes manages scheduling, scaling, self-healing, and traffic routing for you.

---

## 📐 Architecture

```
                         Internet / Browser
                                │
                                ▼
                    ┌────────────────────────┐
                    │   Load Balancer          │  (minikube tunnel /
                    │   (Ingress Controller    │   cloud LB in production)
                    │   Service)                │
                    └────────────────────────┘
                                │
                                ▼
                    ┌────────────────────────┐
                    │   Ingress (nginx)        │  routes by path
                    └────────────────────────┘
                       │                  │
                "/"    │                  │  "/api"
                       ▼                  ▼
            ┌──────────────┐      ┌──────────────┐
            │ Frontend      │      │ Backend       │
            │ Deployment    │      │ Deployment     │
            │ (2 replicas)  │      │ (2 replicas)   │
            └──────────────┘      └──────┬───────┘
                                          │
                                          ▼
                                  ┌──────────────┐
                                  │ Postgres      │
                                  │ (StatefulPod  │
                                  │  + PVC)       │
                                  └──────────────┘

        Cluster-wide:  Prometheus (metrics) + Grafana (dashboards)
```

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| Orchestration | Kubernetes (Minikube) |
| Ingress / Load Balancing | ingress-nginx |
| Application | React frontend, Node.js backend, PostgreSQL (from Project 2) |
| Monitoring | Prometheus + Grafana (via Helm) |
| Manifests | Raw Kubernetes YAML |

---

## 📁 Project Structure

```
kubernetes-production-deployment/
└── k8s/
    ├── 00-namespace.yaml
    ├── 01-postgres-secret.yaml
    ├── 02-postgres-init-configmap.yaml
    ├── 03-postgres-pvc.yaml
    ├── 04-postgres-deployment.yaml
    ├── 05-postgres-service.yaml
    ├── 06-backend-deployment.yaml
    ├── 07-backend-service.yaml
    ├── 08-frontend-deployment.yaml
    ├── 09-frontend-service.yaml
    └── 10-ingress.yaml
```

---

## ✅ Prerequisites

Install these (all free):
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (you already have this)
- [kubectl](https://kubernetes.io/docs/tasks/tools/) — the Kubernetes CLI
- [Minikube](https://minikube.sigs.k8s.io/docs/start/) — runs a local Kubernetes cluster
- [Helm](https://helm.sh/docs/intro/install/) — used to install the monitoring stack

---

## 🚀 Step-by-Step Setup

### 1. Start Minikube and enable Ingress

```bash
minikube start
minikube addons enable ingress
```

Wait for it to confirm the ingress controller is running:
```bash
kubectl get pods -n ingress-nginx
```

### 2. Push your Project 2 images to Docker Hub

If you haven't already pushed the frontend/backend images from Project 2:

```bash
cd ../dockerized-multi-service-app

docker build -t YOUR_DOCKERHUB_USERNAME/multi-service-backend:latest ./backend
docker build -t YOUR_DOCKERHUB_USERNAME/multi-service-frontend:latest ./frontend

docker push YOUR_DOCKERHUB_USERNAME/multi-service-backend:latest
docker push YOUR_DOCKERHUB_USERNAME/multi-service-frontend:latest
```

### 3. Update the image references

In `06-backend-deployment.yaml` and `08-frontend-deployment.yaml`, replace `YOUR_DOCKERHUB_USERNAME` with your actual Docker Hub username:

```bash
cd k8s
sed -i 's/YOUR_DOCKERHUB_USERNAME/your-actual-username/' 06-backend-deployment.yaml 08-frontend-deployment.yaml
```

### 4. Apply all manifests

```bash
kubectl apply -f k8s/
```

`kubectl apply -f` on a directory applies every YAML file inside it — no need to run each one individually.

### 5. Watch everything come up

```bash
kubectl get all -n multiservice
```

Wait until all pods show `Running` and `1/1` or `2/2` ready. Check details on any stuck pod with:
```bash
kubectl describe pod <pod-name> -n multiservice
kubectl logs <pod-name> -n multiservice
```

### 6. Map the Ingress hostname locally

Get Minikube's IP:
```bash
minikube ip
```

Add this line to your hosts file (`C:\Windows\System32\drivers\etc\hosts` on Windows — edit as Administrator):
```
<minikube-ip>  multiservice.local
```

### 7. Open a tunnel (this is your Load Balancer layer)

In a **separate terminal**, keep this running:
```bash
minikube tunnel
```

This gives the ingress controller's Service (type `LoadBalancer`) a real, reachable external IP — exactly how a cloud load balancer would work in production.

### 8. Visit the app

```
http://multiservice.local
```

You should see the same React app from Project 2 — now served entirely from a Kubernetes cluster.

---

## 📊 Adding Monitoring (Prometheus + Grafana)

### 1. Install the kube-prometheus-stack via Helm

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

This single chart installs Prometheus, Grafana, Alertmanager, and exporters that automatically monitor your entire cluster — nodes, pods, deployments — no extra config needed.

### 2. Access Grafana

```bash
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

Visit `http://localhost:3000`.

Get the admin password (it's auto-generated, not a fixed default):
```bash
kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode
```
Username is `admin`.

### 3. Explore the dashboards

Go to **Dashboards** in the Grafana sidebar — you'll already have pre-built dashboards like:
- **Kubernetes / Compute Resources / Namespace (Pods)** — pick `multiservice` to see your app's CPU/memory in real time
- **Kubernetes / Compute Resources / Cluster** — overall cluster health

---

## 🧪 Verifying & Debugging

```bash
# See everything in your namespace
kubectl get all -n multiservice

# Logs for a specific deployment
kubectl logs deployment/backend -n multiservice

# Exec into a pod
kubectl exec -it deployment/postgres -n multiservice -- psql -U postgres -d appdb -c "SELECT * FROM items;"

# Describe a pod stuck in Pending/CrashLoopBackOff
kubectl describe pod <pod-name> -n multiservice

# Restart a deployment without changing anything (forces fresh pods)
kubectl rollout restart deployment/backend -n multiservice
```

---

## 💡 What This Project Demonstrates

- Writing and organizing raw Kubernetes manifests (Namespace, Secret, ConfigMap, PVC, Deployment, Service, Ingress)
- Reusing one Secret across two different containers under different env var names via `secretKeyRef`
- Ingress-based path routing — one entry point splitting traffic to two services
- The Load Balancer layer in front of an Ingress controller (`minikube tunnel`, or a real LoadBalancer Service in cloud Kubernetes)
- Persistent storage for stateful workloads (PostgreSQL) using PersistentVolumeClaims
- Readiness/liveness probes controlling traffic routing and pod restarts
- Cluster-wide observability with Prometheus + Grafana, installed via Helm

---

## 🚀 Possible Extensions

- Add a `HorizontalPodAutoscaler` so the backend scales automatically under load
- Add TLS with `cert-manager` for HTTPS on the Ingress
- Package this app itself as a Helm chart instead of raw manifests
- Add a custom `/metrics` endpoint to the backend (using `prom-client`) and a `ServiceMonitor` so Prometheus scrapes app-specific metrics, not just cluster metrics
- Move from Minikube to a real cloud Kubernetes cluster (EKS/GKE/AKS) — the manifests don't need to change

---

## 👤 Author

**Shawn Sreeju Sampath**
[GitHub](https://github.com/Hacker3S) · [LinkedIn](https://linkedin.com/in/shawn-sreeju-sampath-074923377)

---

## 📄 License

This project is licensed under the MIT License.
