# StackEye GitOps

GitOps repository for StackEye Kubernetes deployments, managed by ArgoCD.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ Developer pushes tag: v1.2.3 or v1.2.3-staging or v1.2.3-prod  │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ GitHub Actions (stackeye/ or stackeye-web/)                     │
│ 1. Build Docker image                                           │
│ 2. Push to Harbor: stackeye/api:v1.2.3                         │
│ 3. Update this repo with new image tag                         │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ This Repository (stackeye-gitops)                               │
│ └── environments/                                               │
│     ├── dev/values.yaml       ← v1.2.3 (default tags)          │
│     ├── staging/values.yaml   ← v1.2.3-staging tags            │
│     └── prod/values.yaml      ← v1.2.3-prod tags               │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ ArgoCD (watches this repo)                                      │
│ - Detects value changes                                         │
│ - Syncs Kubernetes resources                                    │
│ - Provides rollback and drift detection                        │
└─────────────────────────────────────────────────────────────────┘
```

## Repository Structure

```
stackeye-gitops/
├── apps/
│   ├── project.yaml          # ArgoCD AppProject
│   ├── stackeye-api.yaml     # API Applications (dev, staging, prod)
│   ├── stackeye-worker.yaml  # Worker Applications
│   └── stackeye-web.yaml     # Web Applications
├── argocd/
│   └── values.yaml           # ArgoCD installation config
├── environments/
│   ├── dev/
│   │   └── values.yaml       # Dev image tags
│   ├── staging/
│   │   └── values.yaml       # Staging image tags
│   └── prod/
│       └── values.yaml       # Prod image tags
└── README.md
```

## Tag Convention

| Tag Pattern | Target Environment | Description |
|-------------|-------------------|-------------|
| `v1.2.3` | dev | Auto-deploys to development |
| `v1.2.3-staging` | staging | Deploys to staging |
| `v1.2.3-prod` | prod | Deploys to production |

## Prerequisites

- Kubernetes 1.28+
- Helm 3.14+
- ArgoCD 2.10+

## ArgoCD Installation

```bash
# Add ArgoCD Helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install ArgoCD
helm install argocd argo/argo-cd -n argocd --create-namespace \
  -f argocd/values.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## Bootstrap Applications

After ArgoCD is installed, apply the project and applications:

```bash
# Create the StackEye project
kubectl apply -f apps/project.yaml

# Create all applications
kubectl apply -f apps/stackeye-api.yaml
kubectl apply -f apps/stackeye-worker.yaml
kubectl apply -f apps/stackeye-web.yaml
```

## Deployment Workflow

### Deploy to Dev (automatic on any v*.*.* tag)

```bash
cd stackeye/  # or stackeye-web/
git tag v1.2.3
git push origin v1.2.3
```

### Promote to Staging

```bash
git tag v1.2.3-staging
git push origin v1.2.3-staging
```

### Promote to Production

```bash
git tag v1.2.3-prod
git push origin v1.2.3-prod
```

## Manual Operations

### Force Sync

```bash
argocd app sync stackeye-api-prod
```

### Rollback

```bash
argocd app rollback stackeye-api-prod
```

### View Application Status

```bash
argocd app list
argocd app get stackeye-api-prod
```

## Related Repositories

| Repository | Purpose |
|------------|---------|
| [stackeye](https://github.com/StackEye-IO/stackeye) | Backend API + Worker |
| [stackeye-web](https://github.com/StackEye-IO/stackeye-web) | Frontend dashboard |
| [stackeye-deploy](https://github.com/StackEye-IO/stackeye-deploy) | Helm charts |

## License

Apache 2.0
