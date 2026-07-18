# ☸️ TikTo GitOps Manifests

Repository quản lý declarative Kubernetes state cho TikTo, được Argo CD sync tự động lên 2 cluster: **k3s** (dev) và **EKS** (prod).

---

## 🔄 GitOps Delivery Flow

```mermaid
flowchart LR
    AppRepo[TikTo App Repo] --> CD[GitHub Actions CD]
    CD --> Image[Build and Scan Image]
    Image --> GHCR[Push to GHCR]
    CD --> Patch[Update patch-image.yaml]
    Patch --> GitOps[Commit to GitOps Repo]
    GitOps --> Argo[Argo CD Sync]
    Argo --> K8s[EKS / k3s Cluster]
    K8s --> Rollout[Argo Rollout Canary & Analysis]
```

---

## 📂 Repository Layout

```text
.
├── apps/
│   └── tikto/
│       ├── base/                         # Base manifests dùng chung cho tất cả env
│       │   ├── deployment.yaml           # Deployment: tikto (web)
│       │   ├── service.yaml              # Service: tikto (web)
│       │   ├── api-deployment.yaml       # Deployments: profile/tasks/calendar/dashboard API
│       │   └── api-service.yaml          # Services: profile/tasks/calendar/dashboard/gateway API
│       └── overlays/
│           ├── dev/                      # k3s dev environment
│           │   ├── kustomization.yaml
│           │   ├── namespace.yaml
│           │   ├── configmap.yaml
│           │   ├── external-secret.yaml
│           │   ├── secret-store.yaml
│           │   ├── secret.example.yaml   # Cấu trúc secret cần tạo trên AWS
│           │   ├── patch-image.yaml      # Image tags (auto-updated by CI/CD)
│           │   ├── patch-replicas.yaml
│           │   └── patch-service.yaml    # Override sang NodePort cho dev
│           └── prod/                     # EKS prod environment
│               ├── kustomization.yaml
│               ├── namespace.yaml
│               ├── configmap.yaml
│               ├── external-secret.yaml
│               ├── secret-store.yaml
│               ├── secret.example.yaml   # Cấu trúc secret cần tạo trên AWS
│               ├── patch-image.yaml      # Image tags (auto-updated by CI/CD)
│               ├── patch-replicas.yaml
│               ├── patch-service.yaml
│               ├── patch-scheduling.yaml # TopologySpread + PodAffinity rules
│               ├── rollout.yaml          # Argo Rollout: tikto (web) canary
│               ├── rollouts-backend.yaml # Argo Rollout: tikto-gateway canary
│               ├── services-canary.yaml          # tikto-canary / tikto-stable
│               ├── services-backend-canary.yaml  # gateway-canary/stable + VirtualService
│               ├── analysis-smoke-test.yaml      # Web smoke test (200 curl requests)
│               ├── analysis-gateway-smoke.yaml   # Gateway smoke test
│               ├── analysis-opensearch.yaml      # OpenSearch error log check
│               ├── pdb.yaml                      # PodDisruptionBudgets (5 services)
│               ├── istio-gateway.yaml
│               ├── istio-virtualservice.yaml
│               └── ingress.yaml                  # ALB Ingress → Istio ingress
└── argocd/
    ├── server-nodeport.yaml              # Expose ArgoCD UI qua NodePort :30080
    └── applications/
        ├── tikto-dev.yaml               # Argo CD App: dev (k3s-dev cluster)
        ├── tikto-prod.yaml              # Argo CD App: prod (eks-prod cluster)
        └── infra/                       # Infrastructure Helm chart apps
            ├── external-secrets.yaml
            ├── aws-load-balancer-controller.yaml
            ├── istio.yaml               # 3 apps: istio-base + istiod + istio-ingress
            └── fluent-bit.yaml
```

---

## 🏗️ Cluster Bootstrap — Hướng dẫn setup từ đầu

> Thực hiện **theo thứ tự** các bước dưới đây. Mỗi bước là prerequisite của bước sau.

---

### Bước 0 — Chuẩn bị AWS

#### IAM Permissions cho EKS Node Role

Node role của EKS cần được gắn permission sau (qua IRSA hoặc gắn thẳng vào node instance profile):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:ap-southeast-1:*:secret:tikto/*"
    },
    {
      "Effect": "Allow",
      "Action": ["es:ESHttpPost", "es:ESHttpPut"],
      "Resource": "arn:aws:es:ap-southeast-1:*:domain/tikto-prod-logs/*"
    }
  ]
}
```

Ngoài ra cần gắn managed policy cho AWS Load Balancer Controller:

```bash
# Tải policy document
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

# Tạo IAM policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# Gắn vào node role
aws iam attach-role-policy \
  --role-name <EKS_NODE_ROLE_NAME> \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy
```

#### Tag EKS Subnets

ALB cần subnet được tag đúng để biết deploy vào đâu:

```bash
# Public subnets (cho internet-facing ALB)
aws ec2 create-tags --resources <SUBNET_ID> --tags \
  Key=kubernetes.io/cluster/<EKS_CLUSTER_NAME>,Value=shared \
  Key=kubernetes.io/role/elb,Value=1

# Private subnets (cho internal ALB, nếu dùng)
aws ec2 create-tags --resources <SUBNET_ID> --tags \
  Key=kubernetes.io/cluster/<EKS_CLUSTER_NAME>,Value=shared \
  Key=kubernetes.io/role/internal-elb,Value=1
```

#### Tạo Secret trên AWS Secrets Manager

Secret path: **`tikto/secrets`** | Region: **`ap-southeast-1`**

```bash
aws secretsmanager create-secret \
  --name tikto/secrets \
  --region ap-southeast-1 \
  --secret-string '{
    "DATABASE_URL": "postgresql://user:password@host:5432/database",
    "TIKTO_INTERNAL_API_KEY": "your-shared-internal-api-key",
    "NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY": "your-supabase-anon-key",
    "GITOPS_USERNAME": "your-github-username",
    "GITOPS_TOKEN": "your-github-pat-with-packages-read"
  }'
```

> - `GITOPS_USERNAME` + `GITOPS_TOKEN`: dùng để pull image từ `ghcr.io/flavoriy/*`. Token cần scope **`read:packages`**.
> - Tham khảo cấu trúc đầy đủ tại `apps/tikto/overlays/prod/secret.example.yaml`.

---

### Bước 1 — Cài Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Đợi ArgoCD sẵn sàng
kubectl wait --for=condition=available deployment/argocd-server \
  -n argocd --timeout=120s

# Expose ArgoCD UI qua NodePort :30080 (chạy lệnh này trên cluster có ArgoCD)
kubectl apply -f argocd/server-nodeport.yaml

# Lấy mật khẩu admin mặc định
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

> Truy cập UI tại: `https://<node-ip>:30080`
> Đăng nhập: user `admin`, password từ lệnh trên.

---

### Bước 2 — Đăng nhập CLI và register clusters

```bash
# Login ArgoCD CLI
argocd login <node-ip>:30080 --username admin --insecure

# Register k3s dev cluster
argocd cluster add k3s-dev --name k3s-dev

# Register EKS prod cluster
argocd cluster add eks-prod --name eks-prod

# Verify
argocd cluster list
```

> Nếu ArgoCD chạy cùng cluster với app (in-cluster), Application spec đã dùng `server: https://kubernetes.default.svc` — không cần register thêm.

---

### Bước 3 — Cài Argo Rollouts

Cần cài trên **cả hai cluster** (dev và prod nếu dùng Rollout):

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# (Tùy chọn) Cài kubectl plugin để dùng lệnh kubectl argo rollouts
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

---

### Bước 4 — Deploy Infrastructure Helm Charts (EKS prod)

Các Helm chart được quản lý qua Argo CD Application. Sync-wave annotation đảm bảo thứ tự đúng:

| Wave | App | Chart | Namespace |
|------|-----|-------|-----------|
| `-3` | `istio-base` | `istio/base` v1.26.2 | `istio-system` |
| `-3` | `external-secrets` | `external-secrets/external-secrets` v0.14.4 | `external-secrets` |
| `-2` | `istiod` | `istio/istiod` v1.26.2 | `istio-system` |
| `-2` | `aws-load-balancer-controller` | `eks/aws-load-balancer-controller` v1.13.4 | `kube-system` |
| `-1` | `istio-ingress` | `istio/gateway` v1.26.2 | `istio-ingress` |
| `-1` | `fluent-bit` | `fluent/fluent-bit` v0.49.3 | `logging` |

```bash
# ⚠️ Cập nhật clusterName trước khi apply!
# Sửa argocd/applications/infra/aws-load-balancer-controller.yaml:
#   clusterName: <TÊN_EKS_CLUSTER_THỰC_TẾ>

# Apply tất cả — sync-wave tự xử lý thứ tự
kubectl apply -f argocd/applications/infra/

# Theo dõi trạng thái
watch kubectl get applications -n argocd
```

> **Chờ tất cả app Healthy** trước khi sang Bước 5.

**Verify từng component:**

```bash
# External Secrets Operator
kubectl get pods -n external-secrets

# Istio
kubectl get pods -n istio-system
kubectl get pods -n istio-ingress

# AWS LBC
kubectl get pods -n kube-system | grep aws-load-balancer

# Fluent Bit
kubectl get pods -n logging
```

---

### Bước 5 — Deploy Tikto Application

```bash
# Dev environment (k3s)
kubectl apply -f argocd/applications/tikto-dev.yaml

# Prod environment (EKS)
kubectl apply -f argocd/applications/tikto-prod.yaml

# Kiểm tra
argocd app get tikto-prod
argocd app get tikto-dev

# Force sync nếu cần
argocd app sync tikto-prod
```

---

## ⚙️ Cấu hình môi trường

### So sánh Dev vs Prod

| Config | Dev (k3s) | Prod (EKS) |
|--------|-----------|------------|
| **Namespace** | `tikto-dev` | `tikto-prod` |
| **Cluster** | `k3s-dev` | `eks-prod` |
| **Web Service** | `NodePort :30443` | `ClusterIP` (via ALB → Istio) |
| **Gateway Service** | `NodePort :30080` | `ClusterIP` (via Istio VirtualService) |
| **Argo Rollouts** | ❌ Deployment thường | ✅ Rollout canary |
| **Istio sidecar** | ❌ | ✅ (`istio-injection: enabled`) |
| **ALB Ingress** | ❌ | ✅ |
| **PodDisruptionBudget** | ❌ | ✅ |
| **TopologySpread** | ❌ | ✅ |
| **Web replicas** | 1 | 3 |
| **APP_ENV** | `dev` | `prod` |
| **NEXT_PUBLIC_APP_URL** | `http://10.0.1.12:30443` | `http://18.136.110.57:30443` |
| **NEXT_PUBLIC_SUPABASE_URL** | `https://replace-dev-project.supabase.co` | `https://replace-prod-project.supabase.co` |

### Các giá trị cần thay đổi theo môi trường

**`apps/tikto/overlays/prod/configmap.yaml`**

```yaml
NEXT_PUBLIC_APP_URL: http://<EKS_NODE_PUBLIC_IP_HOAC_ALB_DOMAIN>:30443
NEXT_PUBLIC_SUPABASE_URL: https://<PROD_SUPABASE_PROJECT_REF>.supabase.co
```

**`apps/tikto/overlays/dev/configmap.yaml`**

```yaml
NEXT_PUBLIC_APP_URL: http://<K3S_NODE_IP>:30443
NEXT_PUBLIC_SUPABASE_URL: https://<DEV_SUPABASE_PROJECT_REF>.supabase.co
```

**`argocd/applications/infra/aws-load-balancer-controller.yaml`**

```yaml
clusterName: <TÊN_EKS_CLUSTER>   # Phải khớp chính xác với tên cluster trên AWS
```

### Cấu trúc AWS Secrets Manager

Key path: **`tikto/secrets`** (tất cả keys trong 1 secret JSON object)

| Key | Mô tả |
|-----|-------|
| `DATABASE_URL` | PostgreSQL connection string |
| `TIKTO_INTERNAL_API_KEY` | Shared API key giữa các microservice |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | Supabase anon/public key |
| `GITOPS_USERNAME` | GitHub username để pull GHCR image |
| `GITOPS_TOKEN` | GitHub PAT (scope: `read:packages`) |

---

## 🚀 Argo Rollouts & Canary Strategy (Prod only)

### Kiến trúc traffic routing

```
Internet
    │
    ▼
AWS ALB (tikto-ingress) ── healthcheck: :15021/healthz/ready (istio)
    │
    ▼
istio-ingress (NodePort :30443)
    │
    ▼
Istio VirtualService: tikto-virtual-service
    ├── tikto-stable  (weight 100% → giảm dần khi canary)
    └── tikto-canary  (weight 0% → tăng dần: 10→50→80→100%)
    │
    ▼
Rollout: tikto (3 replicas) ── ghcr.io/flavoriy/tikto-web:<version>

─────────────────────────────────────────────────────────

Trong cluster
    │
    ▼
Istio VirtualService: tikto-gateway-vs
    ├── tikto-gateway-stable  (weight 100% → giảm)
    └── tikto-gateway-canary  (weight 0% → tăng)
    │
    ▼
Rollout: tikto-gateway (1 replica) ── ghcr.io/flavoriy/tikto-gateway:<version>
```

### Canary steps (hoàn toàn tự động)

```
Step 1: setWeight 10%  → chờ 1 phút (Analysis chạy)
Step 2: setWeight 50%  → chờ 1 phút (Analysis chạy)
Step 3: setWeight 80%  → chờ 1 phút (Analysis chạy)
Step 4: 100% promote   → Rollout complete ✅
                        hoặc Rollback tự động nếu Analysis fail ❌
```

### Analysis Templates chạy song song

**1. Smoke Test** (`analysis-smoke-test.yaml` / `analysis-gateway-smoke.yaml`)
- Spawn một Kubernetes Job pod
- Chờ canary service ready (max 5 phút, thử mỗi 2 giây)
- Gửi 200 curl requests đến canary endpoint
- `failureLimit: 0` → bất kỳ request nào fail = rollback ngay
- Endpoints: `http://tikto-canary:80/api/health` và `http://tikto-gateway-canary:4000/health`

**2. OpenSearch Error Check** (`analysis-opensearch.yaml`)
- Query OpenSearch mỗi 1 phút
- Tìm log có keyword `error`, `failed`, `exception` từ canary pod (match theo `rollouts-pod-template-hash`)
- Trong khoảng thời gian 2 phút gần nhất
- `successCondition: result.hits.total.value < 5` → nếu ≥ 5 logs lỗi = rollback

### Test canary thủ công (không ảnh hưởng stable traffic)

```bash
# Gửi request trực tiếp đến canary qua header
curl -H "x-canary: true" http://<gateway-endpoint>:4000/health
```

---

## 🛠️ Useful Commands

### Render manifest locally (không cần apply)

```bash
kubectl kustomize apps/tikto/overlays/dev
kubectl kustomize apps/tikto/overlays/prod
```

### Theo dõi Rollout

```bash
# Realtime watch
kubectl argo rollouts get rollout tikto -n tikto-prod --watch
kubectl argo rollouts get rollout tikto-gateway -n tikto-prod --watch

# Xem lịch sử các lần deploy
kubectl argo rollouts history rollout tikto -n tikto-prod

# Xem Istio weights đang được set
kubectl get virtualservice tikto-virtual-service -n tikto-prod -o yaml | grep weight
kubectl get virtualservice tikto-gateway-vs -n tikto-prod -o yaml | grep weight
```

### Manual override Rollout

```bash
# Promote thủ công (skip các pause steps còn lại)
kubectl argo rollouts promote tikto -n tikto-prod
kubectl argo rollouts promote tikto-gateway -n tikto-prod

# Abort và rollback ngay về stable
kubectl argo rollouts abort tikto -n tikto-prod
kubectl argo rollouts abort tikto-gateway -n tikto-prod

# Retry sau khi abort (để tiếp tục từ đầu)
kubectl argo rollouts retry rollout tikto -n tikto-prod
```

### Kiểm tra External Secrets

```bash
# Trạng thái sync (phải là SecretSynced)
kubectl get externalsecret -n tikto-prod
kubectl describe externalsecret tikto-secret -n tikto-prod

# Secret đã được tạo chưa
kubectl get secret tikto-secret -n tikto-prod
kubectl get secret ghcr-secret -n tikto-prod
```

### Kiểm tra Istio

```bash
# Verify sidecar injection — mỗi pod phải có 2 containers
kubectl get pods -n tikto-prod -o jsonpath=\
'{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.name}{","}{end}{"\n"}{end}'

# Xem VirtualService routing (weights thay đổi theo rollout)
kubectl get virtualservice -n tikto-prod

# Proxy config của một pod
istioctl proxy-config routes <pod-name> -n tikto-prod
```

### Kiểm tra ALB / Ingress

```bash
# Xem ALB đã được tạo chưa và DNS của nó
kubectl get ingress tikto-ingress -n tikto-prod

# Xem events nếu ALB chưa được tạo
kubectl describe ingress tikto-ingress -n tikto-prod
```

### Logs

```bash
# Log app container
kubectl logs -n tikto-prod deployment/tikto-calendar-api -c calendar-api -f

# Log Istio sidecar
kubectl logs -n tikto-prod <pod-name> -c istio-proxy -f

# Log Argo Rollouts controller
kubectl logs -n argo-rollouts deployment/argo-rollouts -f

# Log AWS LBC
kubectl logs -n kube-system deployment/aws-load-balancer-controller -f

# Log External Secrets Operator
kubectl logs -n external-secrets deployment/external-secrets -f

# Log Fluent Bit (trên node cụ thể)
kubectl logs -n logging daemonset/fluent-bit -f
```

### Apply Argo CD Applications thủ công

```bash
kubectl apply -f argocd/applications/tikto-dev.yaml
kubectl apply -f argocd/applications/tikto-prod.yaml
kubectl apply -f argocd/applications/infra/
```

---

## 🔍 Troubleshooting

### ❌ ExternalSecret không sync — Status: SecretSyncedError

**Kiểm tra**:
```bash
kubectl describe externalsecret tikto-secret -n tikto-prod
# Xem phần Events và Status.Conditions
```

**Nguyên nhân phổ biến & fix**:
- **Node IAM thiếu permission**: Gắn thêm `secretsmanager:GetSecretValue` vào node role
- **Secret path sai**: Phải là chính xác `tikto/secrets` (phân biệt hoa thường), region `ap-southeast-1`
- **ESO chưa cài**: `kubectl get pods -n external-secrets`
- **SecretStore chưa ready**: `kubectl get secretstore -n tikto-prod`

---

### ❌ Pods ImagePullBackOff — không pull được image từ GHCR

**Nguyên nhân**: `ghcr-secret` chưa được tạo hoặc ExternalSecret chưa sync.

**Fix tạm thời** (khi ESO chưa hoạt động):
```bash
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<GITHUB_USERNAME> \
  --docker-password=<GITHUB_PAT_read:packages> \
  -n tikto-prod
```

---

### ❌ Rollout stuck — không tự tiến qua các steps

**Kiểm tra**:
```bash
kubectl argo rollouts get rollout tikto -n tikto-prod
# Tìm dòng Status và Message

# Xem Analysis runs
kubectl get analysisrun -n tikto-prod
kubectl describe analysisrun <analysisrun-name> -n tikto-prod

# Xem log smoke test job
kubectl get jobs -n tikto-prod
kubectl logs job/<smoke-test-job-name> -n tikto-prod
```

**Fix**:
```bash
# Nếu analysis fail do môi trường (không phải lỗi app): abort rồi retry
kubectl argo rollouts abort tikto -n tikto-prod
kubectl argo rollouts retry rollout tikto -n tikto-prod

# Nếu muốn skip analysis và promote thẳng
kubectl argo rollouts promote tikto -n tikto-prod --full
```

---

### ❌ OpenSearch Analysis trả về lỗi HTTP 400/405

**Nguyên nhân**: Web metric provider cần `method: POST` khi gửi JSON DSL query.

**Đã được fix** trong `analysis-opensearch.yaml`:
```yaml
provider:
  web:
    method: POST        # BẮT BUỘC — mặc định là GET sẽ bị 405
    headers:
      - key: Content-Type
        value: application/json
    jsonBody:
      query: ...
```

Nếu vẫn lỗi, kiểm tra:
- OpenSearch domain endpoint có đúng không
- Node có quyền `es:ESHttpPost` không

---

### ❌ Istio sidecar không được inject vào pod

**Kiểm tra**:
```bash
kubectl get namespace tikto-prod --show-labels
# Phải có: istio-injection=enabled
```

**Fix**: Label đã có trong `namespace.yaml`. Nếu pod chạy trước khi label được thêm, cần restart:
```bash
kubectl rollout restart deployment -n tikto-prod
```

---

### ❌ ALB không được tạo sau khi apply Ingress

**Kiểm tra**:
```bash
# AWS LBC đang chạy chưa
kubectl get pods -n kube-system | grep aws-load-balancer

# Events của Ingress
kubectl describe ingress tikto-ingress -n tikto-prod

# Log LBC
kubectl logs -n kube-system deployment/aws-load-balancer-controller -f
```

**Nguyên nhân phổ biến**:
- `clusterName` trong LBC values chưa khớp với tên EKS cluster thực tế
- Subnet chưa tag `kubernetes.io/role/elb: 1`
- Node IAM thiếu permission `elasticloadbalancing:*`

---

### ❌ Canary Rollout stuck ở Paused (không có duration)

**Nguyên nhân**: Nếu rollout dùng `pause: {}` không có duration thì cần promote thủ công.
Hiện tại tất cả steps đã có `pause: { duration: 1m }` — nếu vẫn stuck hãy kiểm tra analysis.

---

## 📋 Pre-launch Checklist (Prod)

- [ ] Cập nhật `clusterName` trong `argocd/applications/infra/aws-load-balancer-controller.yaml`
- [ ] Tạo secret `tikto/secrets` trên AWS Secrets Manager với đầy đủ 5 keys
- [ ] Tag EKS public subnets với `kubernetes.io/role/elb: 1`
- [ ] Gắn `AWSLoadBalancerControllerIAMPolicy` vào EKS node role
- [ ] Cấp `secretsmanager:GetSecretValue` cho node role
- [ ] Cấp `es:ESHttpPost` cho node role (để Fluent Bit ghi log và Rollout query OpenSearch)
- [ ] Cập nhật `NEXT_PUBLIC_APP_URL` trong `configmap.yaml` với IP/domain thực tế
- [ ] Cập nhật `NEXT_PUBLIC_SUPABASE_URL` với Supabase project URL thực tế
- [ ] Verify ESO sync: `kubectl get externalsecret -n tikto-prod` → `SecretSynced`
- [ ] Verify Istio inject: mỗi pod có 2 containers (`app` + `istio-proxy`)
- [ ] Verify ALB created: `kubectl get ingress -n tikto-prod` → có ADDRESS
- [ ] Test canary flow bằng cách trigger 1 deployment mới
