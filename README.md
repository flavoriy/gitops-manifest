# TikTo GitOps Manifests

Thư mục này chứa các cấu hình Kubernetes của ứng dụng **TikTo**, được quản lý và tự động triển khai bởi **ArgoCD**.

## Cấu trúc thư mục tối giản (Simplified Layout)

* **`apps/tikto/base`**: Các manifest gốc dùng chung cho mọi môi trường (Deployment, Service, Ingress).
* **`apps/tikto/overlays/dev`**: Cấu hình chi tiết cho môi trường **dev** (Namespace `tikto-dev`, NodePort `300443`, cấu hình tự động đồng bộ secret từ AWS).
* **`apps/tikto/overlays/prod`**: Cấu hình chi tiết cho môi trường **prod** (Namespace `tikto-prod`).
* **`argocd/applications`**: Khai báo ArgoCD Applications (`tikto-dev`, `tikto-prod`) để deploy trực tiếp lên cluster.

---

## Cách triển khai ứng dụng qua ArgoCD

Chạy lệnh sau trên cluster để deploy các ứng dụng:

```bash
kubectl apply -f argocd/applications/tikto-dev.yaml
kubectl apply -f argocd/applications/tikto-prod.yaml
```

---

## Quản lý Secrets

Dự án sử dụng **External Secrets Operator (ESO)** để tự động đồng bộ các biến bảo mật từ AWS Secrets Manager (`tikto/dev`) vào trong Kubernetes Secret (`tikto-secret`).
* Không cần tạo secret thủ công.
* Tuyệt đối không commit giá trị mật khẩu/token dạng plain-text lên Git.
