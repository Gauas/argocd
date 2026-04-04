# Gauas ArgoCD GitOps Repository

Multi-environment GitOps automation using Argo CD ApplicationSet.

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

The `applicationset.yaml` uses a **Git Directory Generator** that watches the `clusters/*` pattern.
Whenever a new folder is added under `clusters/`, Argo CD automatically:
1. Detects the new directory
2. Creates a new `Application` pointing to `clusters/<env>/apps/`
3. Syncs all manifests inside that folder to the `<env>` namespace

## Adding a New Environment

```bash
mkdir -p clusters/qa/apps
touch clusters/qa/namespace.yaml   # define Namespace manifest
git add . && git commit -m "feat: add qa environment"
git push origin main
```

That's it. No `kubectl apply` or Argo CD UI interaction needed.

## Removing an Environment

```bash
git rm -r clusters/qa
git commit -m "chore: remove qa environment"
git push origin main
```

Argo CD will automatically prune the `qa-app` Application (because `prune: true` is set).

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

# Sync manually if needed
argocd app sync dev-app
```
