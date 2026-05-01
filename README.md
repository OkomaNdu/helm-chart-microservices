# Online Boutique — Microservices Deployment on Kubernetes

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?style=for-the-badge&logo=helm&logoColor=white)
![Linode](https://img.shields.io/badge/Linode-00A95C?style=for-the-badge&logo=linode&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)
![YAML](https://img.shields.io/badge/YAML-CB171E?style=for-the-badge&logo=yaml&logoColor=white)

> End-to-end deployment of Google's **Online Boutique** — a cloud-native, polyglot microservices e-commerce application — on a production-grade Kubernetes cluster using Helm chart templating and Helmfile for declarative, idempotent release management.

---

## Overview

This project demonstrates the full lifecycle of deploying a distributed microservices application to Kubernetes — from raw manifest authoring and hardening to building reusable Helm chart abstractions and orchestrating multi-release deployments with Helmfile.

The infrastructure runs on **Linode Kubernetes Engine (LKE)** with a 3-node cluster, and all workloads are configured to production standards: pinned image tags, health probes, resource budgets, and minimum replica counts for availability.

**What this project covers:**

- Provisioning a managed Kubernetes cluster on LKE
- Authoring and hardening raw Kubernetes manifests (Deployments + Services)
- Building a **single reusable Helm chart** that serves 10 microservices via value overrides
- Managing a separate **Redis Helm chart** with ephemeral volume persistence
- Orchestrating all 11 releases in a single declarative `helmfile sync` operation
- Applying Kubernetes production best practices across all workloads

---

## Tech Stack

| Layer | Technology |
|---|---|
| Cloud Platform | Linode Kubernetes Engine (LKE) |
| Orchestration | Kubernetes |
| Package Management | Helm v3 |
| Release Orchestration | Helmfile |
| Application Images | Google Container Registry (`gcr.io/google-samples/microservices-demo`) |
| Cart Data Store | Redis (`redis:alpine`) — emptyDir volume |
| Manifest Format | YAML |

---

## Architecture

```
                                ┌─────────────────┐
                                │    INTERNET      │
                                └────────┬─────────┘
                                         │ HTTP :80
                                         ▼

╔════════════════════════════════════════════════════════════════════════════════╗
║  LINODE KUBERNETES ENGINE (LKE)                                               ║
║  3 Worker Nodes  ·  Namespace: microservices                                  ║
║                                                                               ║
║ ┌─────────────────────────────────────────────────────────────────────────┐  ║
║ │  INGRESS TIER                                                            │  ║
║ │                                                                          │  ║
║ │   ┌──────────────────────────────────────────────────────────────────┐  │  ║
║ │   │  frontend                                    type: LoadBalancer   │  │  ║
║ │   │  gcr.io/.../frontend:v0.8.0    port: 80 ──► targetPort: 8080     │  │  ║
║ │   │  replicas: 2  |  liveness: HTTP /_healthz  |  readiness: gRPC    │  │  ║
║ │   └──────────────────────────────────┬───────────────────────────────┘  │  ║
║ └─────────────────────────────────────────────────────────────────────────┘  ║
║                                        │                                      ║
║                              gRPC / HTTP (internal)                           ║
║                                        │                                      ║
║ ┌─────────────────────────────────────────────────────────────────────────┐  ║
║ │  APPLICATION TIER  ·  ClusterIP  ·  2 replicas each  ·  gRPC probes    │  ║
║ │                                                                          │  ║
║ │  ┌────────────────────┐  ┌─────────────────────┐  ┌──────────────────┐ │  ║
║ │  │  checkoutservice   │  │  cartservice         │  │  adservice       │ │  ║
║ │  │  port: 5050        │  │  port: 7070          │  │  port: 9555      │ │  ║
║ │  └────────────────────┘  └──────────┬───────────┘  └──────────────────┘ │  ║
║ │                                     │                                    │  ║
║ │  ┌────────────────────┐  ┌──────────▼──────────┐  ┌──────────────────┐ │  ║
║ │  │  paymentservice    │  │  redis-cart          │  │  emailservice    │ │  ║
║ │  │  port: 50051       │  │  port: 6379          │  │  port: 5000      │ │  ║
║ │  └────────────────────┘  └─────────────────────┘  └──────────────────┘ │  ║
║ │                                                                          │  ║
║ │  ┌────────────────────┐  ┌─────────────────────┐  ┌──────────────────┐ │  ║
║ │  │ productcatalog     │  │  currencyservice     │  │  shippingservice │ │  ║
║ │  │  port: 3550        │  │  port: 7000          │  │  port: 50051     │ │  ║
║ │  └────────────────────┘  └─────────────────────┘  └──────────────────┘ │  ║
║ │                                                                          │  ║
║ │  ┌────────────────────┐                                                  │  ║
║ │  │ recommendationservice                                                 │  ║
║ │  │  port: 8080        │                                                  │  ║
║ │  └────────────────────┘                                                  │  ║
║ └─────────────────────────────────────────────────────────────────────────┘  ║
║                                                                               ║
║ ┌─────────────────────────────────────────────────────────────────────────┐  ║
║ │  DATA TIER                                                               │  ║
║ │                                                                          │  ║
║ │   ┌──────────────────────────────────────────────────────────────────┐  │  ║
║ │   │  redis-cart                                  type: ClusterIP     │  │  ║
║ │   │  redis:alpine  ·  port: 6379  ·  volume: emptyDir → /data        │  │  ║
║ │   └──────────────────────────────────────────────────────────────────┘  │  ║
║ └─────────────────────────────────────────────────────────────────────────┘  ║
╚════════════════════════════════════════════════════════════════════════════════╝

  Note: All services are ClusterIP (internal DNS only) except frontend (LoadBalancer).
        Inter-service discovery via Kubernetes DNS: <service-name>:<port>
```

---

## Repository Structure

```
helm-chart-microservices/
│
├── charts/
│   ├── microservice/               # Reusable Helm chart — shared by all 10 app services
│   │   ├── Chart.yaml
│   │   ├── values.yaml             # Base defaults (overridden per release)
│   │   └── templates/
│   │       ├── deployment.yaml     # Parameterised Deployment
│   │       └── service.yaml        # Parameterised Service
│   │
│   └── redis/                      # Dedicated Helm chart for Redis
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           └── service.yaml
│
├── values/                         # Per-release value overrides
│   ├── ad-service-values.yaml
│   ├── cart-service-values.yaml
│   ├── checkout-service-values.yaml
│   ├── currency-service-values.yaml
│   ├── email-service-values.yaml
│   ├── frontend-values.yaml
│   ├── payment-service-values.yaml
│   ├── productcatalog-service-values.yaml
│   ├── recommendation-service-values.yaml
│   ├── redis-values.yaml
│   └── shipping-service-values.yaml
│
├── helmfile.yaml                   # Declarative multi-release orchestration
├── config.yaml                     # Hardened raw K8s manifests (all 11 services)
├── install.sh                      # helm install — all releases
├── uninstall.sh                    # helm uninstall — all releases
└── Online-Shop-Microservices-kubeconfig.yaml
```

---

## Key Engineering Decisions

### 1. Single Reusable Helm Chart

Rather than maintaining 10 separate charts, a single `charts/microservice` chart is parameterised with `values.yaml`. Each service only supplies what differs — image, port, replicas, env vars — eliminating duplication and ensuring consistency.

```
charts/microservice/   ←── one chart
values/
  ├── frontend-values.yaml
  ├── cartservice-values.yaml
  └── ...                     ←── 10 value files (one override per service)
```

### 2. Helmfile for Idempotent Orchestration

`helmfile sync` deploys or reconciles all 11 Helm releases in a single operation. Subsequent runs are idempotent — only changed releases are updated.

### 3. ClusterIP-only Internal Services

All internal services use `ClusterIP`. `NodePort` is explicitly avoided — it exposes services on every node's external IP, which is a network security risk in production. Only the frontend carries a `LoadBalancer` type for controlled external ingress.

### 4. Health Probes per Protocol

Probe type is matched to each service's protocol rather than applying a generic HTTP check uniformly:

| Service Type | Probe |
|---|---|
| gRPC services | `grpc` probe on the service port |
| Frontend | `httpGet` on `/_healthz` |
| Redis | `tcpSocket` with `initialDelaySeconds: 5` |

### 5. Resource Requests and Limits on All Containers

Every container carries explicit `requests` and `limits`. This prevents resource contention, enables the scheduler to make informed placement decisions, and avoids OOMKilled events under load.

---

## Prerequisites

| Tool | Version | Reference |
|---|---|---|
| `kubectl` | v1.25+ | https://kubernetes.io/docs/tasks/tools/ |
| `helm` | v3.x | https://helm.sh/docs/intro/install/ |
| `helmfile` | v0.150+ | https://github.com/roboll/helmfile |

```bash
kubectl version --client && helm version && helmfile --version
```

---

## Cluster Provisioning — LKE

1. Log in to [Linode Cloud Manager](https://cloud.linode.com)
2. Go to **Kubernetes → Create Cluster**
3. Select region and Kubernetes version
4. Add **3 Worker Nodes** (Shared CPU · 4 GB RAM recommended)
5. Click **Create Cluster** — wait for `Running` status
6. Download the **kubeconfig** from the cluster dashboard

---

## Deployment

### Step 1 — Configure cluster access

```bash
export KUBECONFIG=./Online-Shop-Microservices-kubeconfig.yaml
kubectl get nodes
```

Expected:
```
NAME                     STATUS   ROLES    AGE   VERSION
lke-node-xxxxx-xxxxxx    Ready    <none>   ...   v1.28.x
lke-node-xxxxx-xxxxxx    Ready    <none>   ...   v1.28.x
lke-node-xxxxx-xxxxxx    Ready    <none>   ...   v1.28.x
```

### Step 2 — Create namespace

```bash
kubectl create namespace microservices
kubectl config set-context --current --namespace=microservices
```

### Step 3 — Deploy with Helmfile *(recommended)*

```bash
helmfile sync
```

### Step 3 (alt) — Deploy with Helm individually

```bash
./install.sh
```

Or manually:

```bash
helm install -f values/redis-values.yaml              rediscart              charts/redis
helm install -f values/email-service-values.yaml      emailservice           charts/microservice
helm install -f values/cart-service-values.yaml       cartservice            charts/microservice
helm install -f values/currency-service-values.yaml   currencyservice        charts/microservice
helm install -f values/payment-service-values.yaml    paymentservice         charts/microservice
helm install -f values/recommendation-service-values.yaml recommendationservice charts/microservice
helm install -f values/productcatalog-service-values.yaml productcatalogservice charts/microservice
helm install -f values/shipping-service-values.yaml   shippingservice        charts/microservice
helm install -f values/ad-service-values.yaml         adservice              charts/microservice
helm install -f values/checkout-service-values.yaml   checkoutservice        charts/microservice
helm install -f values/frontend-values.yaml           frontendservice        charts/microservice
```

### Step 3 (alt) — Deploy raw manifests

```bash
kubectl apply -f config.yaml -n microservices
```

### Step 4 — Verify and access

```bash
# Confirm all pods are Running
kubectl get pods -n microservices

# Get the external IP
kubectl get service frontendservice -n microservices
```

Open `http://<EXTERNAL-IP>` in a browser.

---

## Service Configuration

| Service | Container Port | Service Port | Type | Replicas |
|---|---|---|---|---|
| frontend | 8080 | 80 | LoadBalancer | 2 |
| checkoutservice | 5050 | 5050 | ClusterIP | 2 |
| cartservice | 7070 | 7070 | ClusterIP | 2 |
| productcatalogservice | 3550 | 3550 | ClusterIP | 2 |
| currencyservice | 7000 | 7000 | ClusterIP | 2 |
| paymentservice | 50051 | 50051 | ClusterIP | 2 |
| shippingservice | 50051 | 50051 | ClusterIP | 2 |
| emailservice | 8080 | 5000 | ClusterIP | 2 |
| recommendationservice | 8080 | 8080 | ClusterIP | 2 |
| adservice | 9555 | 9555 | ClusterIP | 2 |
| redis-cart | 6379 | 6379 | ClusterIP | 2 |

All app images: `gcr.io/google-samples/microservices-demo/<service>:v0.8.0`
Redis image: `redis:alpine`

---

## Operations

```bash
# View all Helm releases and status
helm list -n microservices

# Watch pod rollout
kubectl get pods -n microservices --watch

# Inspect a pod (events, probe failures, resource usage)
kubectl describe pod <pod-name> -n microservices

# Stream logs
kubectl logs <pod-name> -n microservices -f

# Logs from a previously crashed container
kubectl logs <pod-name> -n microservices --previous

# Preview changes before applying (Helmfile)
helmfile diff

# Upgrade a single release
helm upgrade <release-name> charts/microservice -f values/<service>-values.yaml -n microservices

# View revision history and rollback
helm history <release-name> -n microservices
helm rollback <release-name> <revision> -n microservices

# Exec into a container
kubectl exec -it <pod-name> -n microservices -- /bin/sh
```

---

## Teardown

```bash
# Via Helmfile
helmfile destroy

# Via script
./uninstall.sh

# Via kubectl (raw manifests)
kubectl delete -f config.yaml -n microservices

# Delete the namespace
kubectl delete namespace microservices
```

---

## Production Best Practices Applied

| Practice | Implementation |
|---|---|
| Pinned image versions | All images reference explicit tags — no `latest` |
| Liveness probes | Per-protocol: `grpc`, `httpGet`, `tcpSocket` |
| Readiness probes | Gates traffic until the container is fully ready |
| Resource requests | Guarantees CPU/memory on node scheduling |
| Resource limits | Prevents resource exhaustion across shared nodes |
| No NodePort | All internal services use `ClusterIP` only |
| High availability | Minimum 2 replicas per workload |

---

## References

| Resource | Link |
|---|---|
| Microservices Demo Source | https://github.com/techworld-with-nana/microservices-demo |
| Helm Chart Config Files | https://gitlab.com/twn-devops-bootcamp/latest/10-kubernetes/helm-chart-microservices |
| Helmfile Repo | https://github.com/roboll/helmfile |
| Helm Developer Guide | https://helm.sh/docs/chart_template_guide/ |
| Liveness & Readiness Probes | https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/ |
| Resource Requests & Limits | https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-resource-requests-and-limits |
| K8s Configuration Best Practices | https://kubernetes.io/docs/concepts/configuration/overview/ |
| emptyDir Volumes | https://kubernetes.io/docs/concepts/storage/volumes/#emptydir |
| kubectl Cheat Sheet | https://kubernetes.io/docs/reference/kubectl/cheatsheet/ |
