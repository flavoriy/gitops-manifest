# ☸️ TikTo — GitOps Manifests

This repo manages the **desired state** of Kubernetes for the TikTo application, across both dev and prod-like environments. Argo CD reads this repo and automatically reconciles workloads onto the cluster — it's the single source of truth for what's actually running on K8s.

> 🔗 Related repos: [tikto](https://github.com/flavoriy/tikto) (builds & pushes images, auto-commits into this repo) · [Infrastructure-as-Code](https://github.com/flavoriy/Infrastructure-as-Code) (the underlying EKS infrastructure)

---

## 🔄 GitOps Delivery Flow

```mermaid
flowchart LR
    AppRepo[tikto repo] --> CD[GitHub Actions CD]
    CD --> Image[Build + Scan Image]
    Image --> GHCR[Push to GHCR]
    CD --> Patch[Update patch-image.yaml]
    Patch --> GitOps[Commit into this repo]
    GitOps --> Argo[Argo CD Sync]
    Argo --> K8s[EKS Cluster]
    K8s --> Rollout[Argo Rollouts Canary + Analysis]
```

Key point: **the `tikto` repo never deploys directly**. It only builds/scans/pushes images, then auto-commits the new tag into this repo. Argo CD is the only thing that actually pushes changes onto the cluster — cleanly separating CI (build) from CD (deploy).

---

## 🌱 Two environments, one repo

This repo drives **two separate clusters** from the same Argo CD instance, using two Kustomize overlays (`overlays/dev` and `overlays/prod`). Same application, same base manifests — only the runtime target, ingress path, and delivery strategy differ.

```mermaid
flowchart TB
    User[End User] -->|HTTPS 443| ALB["AWS ALB (internet-facing)"]
    Admin[DevOps Engineer] -->|Tailscale VPN, private only| K3sNode[K3s Node]

    subgraph ProdCluster["EKS Prod — namespace tikto-prod"]
        ALB -->|target: NodePort 31380 on EKS worker| IstioSvc["Service: istio-ingressgateway (type: NodePort)"]
        IstioSvc --> IstioPod[Istio Ingress Gateway Pod]
        IstioPod --> FERollout["Rollout: tikto (frontend canary)"]
        IstioPod --> GWRollout["Rollout: tikto-gateway (canary)"]
        GWRollout --> ProfileP[Deployment: profile]
        GWRollout --> TasksP[Deployment: tasks]
        GWRollout --> CalendarP[Deployment: calendar]
        GWRollout --> DashboardP[Deployment: dashboard]
        FluentProd[Fluent Bit DaemonSet] -.ships logs.-> OS[(AWS OpenSearch)]
        Analysis[AnalysisRun: check-canary-errors] -->|DSL query every cycle| OS
        Analysis --> FERollout
        Analysis --> GWRollout
    end

    subgraph DevCluster["K3s Dev — namespace tikto-dev, single EC2"]
        K3sNode -->|"Service: tikto (type: NodePort 30080)"| DevFE[Deployment: tikto frontend]
        K3sNode -->|"Service: tikto-gateway (type: NodePort 30081)"| DevGW[Deployment: tikto-gateway]
        DevGW --> DevProfile[Deployment: profile]
        DevGW --> DevTasks[Deployment: tasks]
        DevGW --> DevCalendar[Deployment: calendar]
        DevGW --> DevDashboard[Deployment: dashboard]
    end
```

**Which uses NodePort, which uses ALB — and why:**

- **Dev** exposes `tikto` and `tikto-gateway` as plain **NodePort** Services (e.g. `30080`, `30081`). There's no load balancer in front — the only way in is over the Tailscale VPN straight to the K3s node's IP, so a NodePort is enough and cheaper to run.
- **Prod** puts an internet-facing **AWS ALB** in front. Under the hood, the ALB (in *instance* target mode) still targets a **NodePort** on the EKS worker nodes — that NodePort belongs to the `istio-ingressgateway` Service, not the app directly. Istio then routes into whichever Rollout (frontend/gateway) is currently receiving canary traffic.
- Only the frontend and gateway get an ingress path / Rollout at all — `profile`, `tasks`, `calendar`, `dashboard` are internal-only Deployments reached through the gateway, in both environments.

**Centralized logging (Prod only):** a Fluent Bit DaemonSet runs in the EKS cluster and ships pod logs to the **AWS OpenSearch** domain (provisioned in `Infrastructure-as-Code`). This is queried live by the `check-canary-errors` AnalysisRun to decide whether to promote or auto-rollback a canary. Dev has no logging pipeline — it's a throwaway environment for manifest sanity checks, not something worth centralizing logs for.

| | **Dev (`overlays/dev`)** | **Prod (`overlays/prod`)** |
|---|---|---|
| Cluster | K3s, single EC2 node | EKS, Spot Node Group (multi-node) |
| Access | Private, via Tailscale VPN only | Public, via AWS ALB → Istio Ingress (NodePort) |
| Ingress | Direct NodePort per service | ALB → NodePort → Istio Ingress Gateway |
| Rollout strategy | Standard Deployment (no canary) | Argo Rollouts canary with automated Analysis |
| Logging | None | Fluent Bit → OpenSearch, drives auto-rollback |
| Rollback | Manual (`kubectl rollout undo`) | Automatic, triggered by log-based error analysis |
| Purpose | Fast iteration, verify manifests before promoting | Real production traffic |

Both Argo CD Applications point at the **same repo**, just different overlay paths — so a change to `apps/tikto/base` propagates to both environments, while anything under `overlays/dev` or `overlays/prod` stays environment-specific.

---

## 📂 Repository Layout

```
.
├── apps/
│   └── tikto/
│       ├── base/                          # Shared base K8s manifests
│       └── overlays/
│           ├── dev/                       # Dev environment overlay
│           └── prod/                      # Prod environment overlay (EKS)
│               ├── rollout.yaml               # Frontend Canary Rollout
│               ├── rollouts-backend.yaml      # API Gateway Canary Rollout
│               ├── analysis-opensearch.yaml   # OpenSearch log-scanning template
│               └── analysis-gateway-smoke.yaml# API Gateway smoke test
└── argocd/
    └── applications/                       # Argo CD Application manifests
```

Uses **Kustomize** (base + overlays) to avoid duplicating manifests between dev/prod — only the actual differences (image tag, replica count, resource limits...) are patched.

---

## 🚀 Argo Rollouts & Canary Deployment (Prod)

In the `tikto-prod` environment, both the Frontend (`tikto`) and API Gateway (`tikto-gateway`) use **Argo Rollouts** instead of a standard Deployment:

1. **Traffic Splitting**: Istio gradually shifts traffic: `10% → 50% → 80% → 100%`.
2. **Automated Smoke Test (`run-smoke-test`)**: a Job runs 200 curl requests against the canary endpoints (`tikto-canary`, `tikto-gateway-canary:4000/health`) before promoting.
3. **Log-based Analysis (`check-canary-errors`)**: queries AWS OpenSearch in real time via DSL. If the count of logs containing `error`/`failed`/`exception` tied to the canary's hash exceeds **5 within 2 minutes**, the rollout is marked failed and auto-rolled back.

---

## 🛠️ Common Operational Commands

### Apply Argo CD Applications

```bash
kubectl apply -f argocd/applications/tikto-dev.yaml
kubectl apply -f argocd/applications/tikto-prod.yaml
```

### Render manifests locally before applying

```bash
kubectl kustomize apps/tikto/overlays/dev
kubectl kustomize apps/tikto/overlays/prod
```

### Monitor rollout progress

```bash
kubectl argo rollouts get rollout tikto -n tikto-prod
kubectl argo rollouts get rollout tikto-gateway -n tikto-prod
```

### Manual control

```bash
# Manually promote
kubectl argo rollouts promote tikto-gateway -n tikto-prod

# Abort and roll back immediately
kubectl argo rollouts abort tikto-gateway -n tikto-prod
```

---

## 🔍 Troubleshooting

### ❌ OpenSearch metric queries fail

- **Cause**: Argo Rollouts' `web` metric provider defaults to `GET`, but OpenSearch DSL queries require `POST`.
- **Fix**: explicitly set the method in the `AnalysisTemplate`:

```yaml
provider:
  web:
    method: POST  # required for OpenSearch search payloads
    url: https://<opensearch-domain>/kubernetes-logs/_search
    headers:
      - key: Content-Type
        value: application/json
```

### ❌ Canary Rollout stuck in Suspended state

- **Cause**: the `pause: {}` step has no duration, requiring manual intervention to promote.
- **Fix**: always set an explicit duration so progressive delivery stays fully automated:

```yaml
steps:
  - setWeight: 20
  - pause: { duration: 1m }
  - setWeight: 50
  - pause: { duration: 1m }
```

---

## 🧱 Tech stack

`Kubernetes` `Kustomize` `Argo CD` `Argo Rollouts` `Istio` `AWS OpenSearch`
