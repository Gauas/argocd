# Gauas ArgoCD GitOps Repository

Multi-environment GitOps automation using Argo CD ApplicationSet.

Current mode: the ApplicationSet manages 3 environments, each mapped to
its own Git branch:
- `dev` -> branch `dev`
- `staging` -> branch `staging`
- `prod` -> branch `prod`

## Repository Structure

```
argocd/
├── applicationset.yaml          # ApplicationSet - brain of the automation
└── clusters/
    ├── dev/
    │   ├── namespace.yaml       # Namespace definition
    │   └── apps/
    │       └── license-service.yaml
    ├── staging/
    │   ├── namespace.yaml
    │   └── apps/
    │       └── license-service.yaml
    └── prod/
        ├── namespace.yaml
        └── apps/
            └── license-service.yaml
```

## How It Works

The `applicationset.yaml` uses a **List Generator** to define 3
Applications explicitly:
1. `dev-app` reads manifests from branch `dev` at `clusters/dev/`
2. `staging-app` reads manifests from branch `staging` at `clusters/staging/`
3. `prod-app` reads manifests from branch `prod` at `clusters/prod/`

Each Application syncs to its matching namespace:
- `dev-app` -> namespace `dev`
- `staging-app` -> namespace `staging`
- `prod-app` -> namespace `prod`

Because the source path is the whole environment folder, Argo CD applies:
- `namespace.yaml`
- everything under `apps/`

## Adding a New Environment

To add another environment, you need to:
1. Create a branch for that environment
2. Add `clusters/<env>/namespace.yaml`
3. Add manifests under `clusters/<env>/apps/`
4. Add a new element to `applicationset.yaml`

## Removing an Environment

Remove the corresponding element from `applicationset.yaml` and delete
the branch/folder if no longer needed.

## Deploy ApplicationSet

```bash
kubectl apply -f applicationset.yaml
```

## Verify

```bash
# List all generated applications
argocd app list

# Check specific environment
argocd app get dev-app
argocd app get staging-app
argocd app get prod-app

argocd app sync dev-app
argocd app sync staging-app
argocd app sync prod-app
```
