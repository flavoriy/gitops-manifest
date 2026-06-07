# GitOps manifests

This repository is the deployment source watched by ArgoCD.

## Layout

```text
apps/tikto/base                  # Shared Kubernetes manifests
apps/tikto/overlays/dev          # Dev namespace or dev cluster config
apps/tikto/overlays/prod         # Prod namespace or prod cluster config
argocd/projects/tikto.yaml       # ArgoCD project boundary
argocd/applications/tikto-dev.yaml
argocd/applications/tikto-prod.yaml
```

## Before first sync

1. Replace `https://github.com/Flavoriy/gitops-manifest.git` in `argocd/` with the real manifest repository URL.
2. Replace `dev.tikto.example.com` and `tikto.example.com` with real DNS names or local hosts entries.
3. Create the runtime secret in each target namespace. Do not commit real secrets.

```bash
kubectl create namespace tikto-dev
kubectl apply -f apps/tikto/overlays/dev/secret.example.yaml

kubectl create namespace tikto-prod
kubectl apply -f apps/tikto/overlays/prod/secret.example.yaml
```

For real production, replace the example Secret flow with Sealed Secrets, External Secrets, or SOPS.

## ArgoCD

If ArgoCD runs inside each cluster, keep:

```yaml
destination:
  server: https://kubernetes.default.svc
```

If one central ArgoCD controls both dev and prod clusters, register both clusters in ArgoCD and replace the prod/dev `destination.server` values with the registered cluster API server URLs or cluster names.

Install the ArgoCD objects:

```bash
kubectl apply -f argocd/projects/tikto.yaml
kubectl apply -f argocd/applications/tikto-dev.yaml
kubectl apply -f argocd/applications/tikto-prod.yaml
```

## Jenkins image updates

The current Jenkins shared library can update a manifest file by replacing a line that starts with `image: ghcr.io/flavoriy/tikto`.

Use these files:

```text
Dev:  apps/tikto/overlays/dev/patch-image.yaml
Prod: apps/tikto/overlays/prod/patch-image.yaml
```

Recommended flow:

```text
develop -> Jenkins updates dev patch -> ArgoCD syncs dev
main/prod promotion -> create PR updating prod patch -> approval -> ArgoCD syncs prod
```

Prod should promote the exact image tag already tested in dev.

