# Online Boutique — Kubernetes Microservices Deployment

Production-grade deployment of Google's **Online Boutique** e-commerce application on **Linode Kubernetes Engine (LKE)** using Helm charts and Helmfile for declarative, repeatable release management.

---

## Table of Contents

1. [Stack](#stack)
2. [Architecture](#architecture)
3. [Service Topology](#service-topology)
4. [Repository Structure](#repository-structure)
5. [Prerequisites](#prerequisites)
6. [Infrastructure Provisioning — LKE](#infrastructure-provisioning--lke)
7. [Cluster Access & Namespace Setup](#cluster-access--namespace-setup)
8. [Deployment](#deployment)
   - [Option A — Helmfile (Recommended)](#option-a--helmfile-recommended)
   - [Option B — Helm Install (Individual Releases)](#option-b--helm-install-individual-releases)
   - [Option C — Raw Manifests](#option-c--raw-manifests)
9. [Helm Chart Design](#helm-chart-design)
10. [Production Hardening](#production-hardening)
11. [Helm Chart Values Reference](#helm-chart-values-reference)
12. [Service Configuration Matrix](#service-configuration-matrix)
13. [Operations Runbook](#operations-runbook)
14. [Teardown](#teardown)
15. [References](#references)

---

## Stack

| Component | Technology |
|---|---|
| Container Orchestration | Kubernetes (LKE) |
| Package Management | Helm v3 |
| Release Orchestration | Helmfile |
| Cart Persistence | Redis (emptyDir volume) |
| Container Registry | Google Container Registry (`gcr.io`) |
| Cloud Provider | Linode Kubernetes Engine (LKE) |

---

## Architecture

```
                                        ┌───────────────┐
                                        │    INTERNET   │
                                        └───────┬───────┘
                                                │
                                            HTTP :80
                                                │
                                                ▼
╔═══════════════════════════════════════════════════════════════════════════════════╗
║              LINODE KUBERNETES ENGINE (LKE) — 3 Worker Nodes                     ║
║              Namespace: microservices                                             ║
║                                                                                   ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐ ║
║  │                          EXTERNAL LOAD BALANCER                              │ ║
║  │                                                                               │ ║
║  │              ┌─────────────────────────────────────────┐                     │ ║
║  │              │  frontend          type: LoadBalancer    │                     │ ║
║  │              │  port: 80  ──────► targetPort: 8080      │                     │ ║
║  │              │  replicas: 2 · image: .../frontend:v0.8.0│                     │ ║
║  │              └─────────────────────────┬───────────────┘                     │ ║
║  └────────────────────────────────────────┼────────────────────────────────────┘ ║
║                                           │                                       ║
║                              Internal gRPC / HTTP traffic                         ║
║                                           │                                       ║
║  ┌────────────────────────────────────────┼────────────────────────────────────┐ ║
║  │                        APPLICATION TIER (ClusterIP)                          │ ║
║  │                                        │                                     │ ║
║  │          ┌─────────────────────────────┼──────────────────────────────┐     │ ║
║  │          │                             │                              │     │ ║
║  │   ┌──────▼──────────────┐  ┌──────────▼──────────┐  ┌───────────────▼──┐  │ ║
║  │   │   cartservice        │  │   checkoutservice    │  │  adservice        │  │ ║
║  │   │   ClusterIP · :7070  │  │   ClusterIP · :5050  │  │  ClusterIP · :9555│  │ ║
║  │   │   replicas: 2        │  │   replicas: 2        │  │  replicas: 2     │  │ ║
║  │   └──────────┬───────────┘  └──────────┬───────────┘  └──────────────────┘  │ ║
║  │              │                          │                                     │ ║
║  │   ┌──────────▼───────────┐  ┌──────────▼───────────┐                        │ ║
║  │   │   paymentservice     │  │   emailservice        │                        │ ║
║  │   │   ClusterIP · :50051 │  │   ClusterIP · :5000   │                        │ ║
║  │   │   replicas: 2        │  │   replicas: 2         │                        │ ║
║  │   └──────────────────────┘  └───────────────────────┘                        │ ║
║  │                                                                               │ ║
║  │   ┌──────────────────────┐  ┌───────────────────────┐  ┌──────────────────┐  │ ║
║  │   │  productcatalogservice│  │   currencyservice     │  │  shippingservice  │  │ ║
║  │   │  ClusterIP · :3550   │  │   ClusterIP · :7000   │  │  ClusterIP·:50051│  │ ║
║  │   │  replicas: 2         │  │   replicas: 2         │  │  replicas: 2     │  │ ║
║  │   └──────────────────────┘  └───────────────────────┘  └──────────────────┘  │ ║
║  │                                                                               │ ║
║  │   ┌──────────────────────┐                                                   │ ║
║  │   │  recommendationservice│                                                   │ ║
║  │   │  ClusterIP · :8080   │                                                   │ ║
║  │   │  replicas: 2         │                                                   │ ║
║  │   └──────────────────────┘                                                   │ ║
║  └───────────────────────────────────────────────────────────────────────────────┘ ║
║                                                                                   ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐ ║
║  │                           DATA TIER                                          │ ║
║  │                                                                               │ ║
║  │              ┌─────────────────────────────────────────┐                     │ ║
║  │              │  redis-cart         ClusterIP · :6379    │                     │ ║
║  │              │  image: redis:alpine · replicas: 2       │                     │ ║
║  │              │  volume: emptyDir mounted at /data        │                     │ ║
║  │              └─────────────────────────────────────────┘                     │ ║
║  └─────────────────────────────────────────────────────────────────────────────┘ ║
╚═══════════════════════════════════════════════════════════════════════════════════╝

  ╔══╗  = LKE cluster boundary          ┌──┐ = Kubernetes Service / Workload
  ║  ║                                  └──┘
  ╚══╝  External LoadBalancer exposes frontend only. All other services are
        ClusterIP (internal DNS resolution via <service-name>:<port>).
```

---

## Service Topology

Inter-service communication uses Kubernetes internal DNS (`<service-name>:<port>`). Only the frontend exposes a public endpoint.

```
Client
  └──► frontend            [LoadBalancer :80 → :8080]
          ├──► productcatalogservice   [:3550]
          ├──► currencyservice         [:7000]
          ├──► cartservice             [:7070]
          │       └──► redis-cart      [:6379]
          ├──► recommendationservice   [:8080]
          │       └──► productcatalogservice [:3550]
          ├──► shippingservice         [:50051]
          ├──► checkoutservice         [:5050]
          │       ├──► productcatalogservice [:3550]
          │       ├──► shippingservice       [:50051]
          │       ├──► paymentservice        [:50051]
          │       ├──► emailservice          [:5000]
          │       ├──► currencyservice       [:7000]
          │       └──► cartservice           [:7070]
          └──► adservice               [:9555]
```

---

## Repository Structure

```
helm-chart-microservices/
├── charts/
│   ├── microservice/               # Generic reusable chart for all app services
│   │   ├── Chart.yaml
│   │   ├── values.yaml             # Chart defaults (overridden per release)
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       └── service.yaml
│   └── redis/                      # Dedicated chart for Redis cart store
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           └── service.yaml
├── values/                         # Release-specific value overrides
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
├── helmfile.yaml                   # Declarative multi-release manifest
├── config.yaml                     # Raw K8s manifests (best-practices hardened)
├── install.sh                      # Convenience script — helm install all releases
├── uninstall.sh                    # Convenience script — helm uninstall all releases
└── Online-Shop-Microservices-kubeconfig.yaml
```

---

## Prerequisites

| Tool | Required Version | Install Reference |
|---|---|---|
| `kubectl` | v1.25+ | https://kubernetes.io/docs/tasks/tools/ |
| `helm` | v3.x | https://helm.sh/docs/intro/install/ |
| `helmfile` | v0.150+ | https://github.com/roboll/helmfile |

Verify toolchain:

```bash
kubectl version --client
helm version
helmfile --version
```

---

## Infrastructure Provisioning — LKE

Provision a 3-node managed Kubernetes cluster on Linode Kubernetes Engine (LKE):

1. Log in to [Linode Cloud Manager](https://cloud.linode.com)
2. Navigate to **Kubernetes** → **Create Cluster**
3. Select region and Kubernetes version
4. Add **3 Worker Nodes** — Shared CPU, 4GB RAM recommended
5. Click **Create Cluster** and wait for `Running` status
6. Download the cluster **kubeconfig** from the cluster dashboard

---

## Cluster Access & Namespace Setup

```bash
# Point kubectl at the LKE cluster
export KUBECONFIG=./Online-Shop-Microservices-kubeconfig.yaml

# Confirm all nodes are Ready
kubectl get nodes
```

Expected output:
```
NAME                          STATUS   ROLES    AGE   VERSION
lke-node-xxxxx-xxxxxxx        Ready    <none>   ...   v1.28.x
lke-node-xxxxx-xxxxxxx        Ready    <none>   ...   v1.28.x
lke-node-xxxxx-xxxxxxx        Ready    <none>   ...   v1.28.x
```

```bash
# Create a dedicated namespace
kubectl create namespace microservices

# Set as the default namespace for this context
kubectl config set-context --current --namespace=microservices
```

---

## Deployment

### Option A — Helmfile (Recommended)

Helmfile provides idempotent, declarative deployment of all 11 releases in a single operation.

**Install Helmfile:**

```bash
curl -Lo helmfile https://github.com/roboll/helmfile/releases/latest/download/helmfile_linux_amd64
chmod +x helmfile
sudo mv helmfile /usr/local/bin/
```

**Deploy all releases:**

```bash
helmfile sync
```

**Validate rollout:**

```bash
kubectl get pods -n microservices
kubectl get services -n microservices
```

**Retrieve the frontend endpoint:**

```bash
kubectl get service frontendservice -n microservices
# Open http://<EXTERNAL-IP> in a browser
```

---

### Option B — Helm Install (Individual Releases)

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

Or use the convenience script:

```bash
chmod +x install.sh && ./install.sh
```

---

### Option C — Raw Manifests

Deploys via the hardened `config.yaml` manifest directly with `kubectl`. Suitable for environments without Helm.

```bash
kubectl apply -f config.yaml -n microservices

# Verify
kubectl get pods -n microservices
kubectl get services -n microservices
```

---

## Helm Chart Design

A **single reusable Helm chart** (`charts/microservice`) serves all 10 application services. Per-service configuration is injected exclusively through `values/` overrides, eliminating manifest duplication.

**`charts/microservice/templates/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.appName }}
spec:
  replicas: {{ .Values.appReplicas }}
  selector:
    matchLabels:
      app: {{ .Values.appName }}
  template:
    metadata:
      labels:
        app: {{ .Values.appName }}
    spec:
      containers:
      - name: {{ .Values.appName }}
        image: "{{ .Values.appImage }}:{{ .Values.appVersion }}"
        ports:
        - containerPort: {{ .Values.containerPort }}
        env:
        {{- range .Values.containerEnvVars}}
        - name: {{ .name }}
          value: {{ .value | quote }}
        {{- end}}
```

**`charts/microservice/templates/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.appName }}
spec:
  type: {{ .Values.serviceType }}
  selector:
    app: {{ .Values.appName }}
  ports:
  - protocol: TCP
    port: {{ .Values.servicePort }}
    targetPort: {{ .Values.containerPort }}
```

**Validate before deploying:**

```bash
helm lint charts/microservice
helm lint charts/redis

# Render templates locally (no cluster interaction)
helm template frontendservice charts/microservice -f values/frontend-values.yaml
```

---

## Production Hardening

The following production best practices are enforced across all services in both `config.yaml` and the Helm chart values:

### 1. Pinned Image Versions

All containers reference explicit image tags — no `latest`. This guarantees reproducible deployments and prevents unintentional upgrades.

```yaml
image: gcr.io/google-samples/microservices-demo/frontend:v0.8.0
```

### 2. Liveness Probes

Triggers automatic container restart on failure. Probe type matches the service protocol:

```yaml
# gRPC services
livenessProbe:
  grpc:
    port: 8080
  periodSeconds: 5

# Frontend (HTTP)
livenessProbe:
  httpGet:
    path: "/_healthz"
    port: 8080
  periodSeconds: 5

# Redis (TCP)
livenessProbe:
  tcpSocket:
    port: 6379
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 3. Readiness Probes

Gates traffic to a Pod until the application signals it is ready to serve requests.

```yaml
readinessProbe:
  grpc:
    port: 8080
  periodSeconds: 5
```

### 4. Resource Requests

Ensures the scheduler places Pods on nodes with sufficient available capacity.

```yaml
resources:
  requests:
    cpu: 100m
    memory: 64Mi
```

### 5. Resource Limits

Enforces ceiling on CPU and memory consumption, protecting cluster stability.

```yaml
resources:
  limits:
    cpu: 200m
    memory: 128Mi
```

### 6. ClusterIP for All Internal Services

Internal services are exposed only within the cluster via `ClusterIP`. `NodePort` is not used, as it exposes ports on every node's public IP — a network attack surface in production environments.

```yaml
spec:
  type: ClusterIP   # All internal services
```

```yaml
spec:
  type: LoadBalancer   # Frontend only — external traffic entrypoint
```

### 7. Multiple Replicas

All services run a minimum of **2 replicas**, ensuring availability during rolling updates and node failures.

```yaml
spec:
  replicas: 2
```

---

## Helm Chart Values Reference

### `charts/microservice`

| Key | Type | Default | Description |
|---|---|---|---|
| `appName` | string | `servicename` | Name applied to Deployment and Service |
| `appImage` | string | `gcr.io/google-samples/...` | Container image path (without tag) |
| `appVersion` | string | `v0.0.0` | Container image tag |
| `appReplicas` | int | `1` | Number of Pod replicas |
| `containerPort` | int | `8080` | Port the container process binds to |
| `containerEnvVars` | list | `[]` | List of `{name, value}` environment variables |
| `servicePort` | int | `8080` | Port exposed on the Service object |
| `serviceType` | string | `ClusterIP` | Kubernetes Service type |

### `charts/redis`

| Key | Type | Default | Description |
|---|---|---|---|
| `appName` | string | `redis` | Name applied to Deployment and Service |
| `appImage` | string | `redis` | Redis image name |
| `appVersion` | string | `alpine` | Redis image tag |
| `appReplicas` | int | `1` | Number of Pod replicas |
| `containerPort` | int | `6379` | Redis port |
| `volumeName` | string | `redis-data` | emptyDir volume name |
| `containerMountPath` | string | `/data` | Redis data directory mount path |
| `servicePort` | int | `6379` | Port exposed on the Service object |

---

## Service Configuration Matrix

All application images are pulled from `gcr.io/google-samples/microservices-demo/` at tag `v0.8.0`.
Redis is pulled from Docker Hub (`redis:alpine`).

| Service | Container Port | Service Port | Service Type | Replicas |
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

---

## Operations Runbook

### Check release status

```bash
# List all Helm releases and their states
helm list -n microservices

# Watch Pod rollout in real time
kubectl get pods -n microservices --watch

# Inspect a specific Pod (events, resource usage, probe failures)
kubectl describe pod <pod-name> -n microservices
```

### View application logs

```bash
# Stream logs
kubectl logs <pod-name> -n microservices -f

# Retrieve logs from a previously crashed container
kubectl logs <pod-name> -n microservices --previous
```

### Upgrade a release

```bash
helm upgrade <release-name> charts/microservice -f values/<service>-values.yaml -n microservices
```

### Roll back a release

```bash
# View revision history
helm history <release-name> -n microservices

# Roll back to a specific revision
helm rollback <release-name> <revision> -n microservices
```

### Preview changes before applying (Helmfile)

```bash
helmfile diff
```

### Execute into a running container

```bash
kubectl exec -it <pod-name> -n microservices -- /bin/sh
```

### Retrieve the public frontend URL

```bash
kubectl get service frontendservice -n microservices
# EXTERNAL-IP column → http://<EXTERNAL-IP>
```

---

## Teardown

### Via Helmfile

```bash
helmfile destroy
```

### Via Helm (individual releases)

```bash
chmod +x uninstall.sh && ./uninstall.sh
```

Or manually:

```bash
helm uninstall rediscart emailservice cartservice currencyservice paymentservice \
  recommendationservice productcatalogservice shippingservice adservice \
  checkoutservice frontendservice -n microservices
```

### Via kubectl (raw manifest)

```bash
kubectl delete -f config.yaml -n microservices
```

### Delete the namespace

```bash
kubectl delete namespace microservices
```

---

## References

| Resource | URL |
|---|---|
| Microservices Demo Source | https://github.com/techworld-with-nana/microservices-demo |
| Helm Chart Config Files | https://gitlab.com/twn-devops-bootcamp/latest/10-kubernetes/helm-chart-microservices |
| Helmfile Official Repo | https://github.com/roboll/helmfile |
| Helm Chart Developer Guide | https://helm.sh/docs/chart_template_guide/ |
| Helm Built-In Objects | https://helm.sh/docs/chart_template_guide/builtin_objects/ |
| Helm Best Practices | https://helm.sh/docs/chart_best_practices/ |
| Redis Docker Image | https://hub.docker.com/_/redis |
| Kubernetes emptyDir Volume | https://kubernetes.io/docs/concepts/storage/volumes/#emptydir |
| Liveness & Readiness Probes | https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/ |
| Resource Requests & Limits | https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-resource-requests-and-limits |
| K8s Configuration Best Practices | https://kubernetes.io/docs/concepts/configuration/overview/ |
| K8s Security Best Practices | https://www.youtube.com/watch?v=oBf5lrmquYI |
| Istio Service Mesh | https://youtu.be/16fgzklcF7Y |
| kubectl Cheat Sheet | https://kubernetes.io/docs/reference/kubectl/cheatsheet/ |
