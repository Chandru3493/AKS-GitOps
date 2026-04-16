# AKS-GitOps

GitOps repository for managing Kubernetes workloads on Azure AKS using ArgoCD. All cluster state is declared in this repo — pushing a change here automatically reflects in the cluster.

## Overview

```
GitHub (source of truth)
        │
        ▼
   ArgoCD (app-of-apps pattern)
        │
        ├── nginx-ingress      → Ingress controller (public IP: 20.84.25.4)
        ├── argocd-ingress     → ArgoCD self-management + ingress
        ├── prometheus         → kube-prometheus-stack (Prometheus + Grafana)
        ├── prometheus-ingress → Ingress for Prometheus and Grafana
        └── todo-app           → Sample todo microservice
```

## Repository Structure

```
AKS-GitOps/
├── .github/workflows/
│   ├── argocd-install.yaml          # Bootstrap ArgoCD on a fresh cluster (run once)
│   ├── aks-start-stop.yaml          # Manually start or stop the AKS cluster
│   └── aks-scheduled-start-stop.yaml # Auto-stop cluster at 8 scheduled IST times
│
├── argocd/
│   ├── app-of-apps.yaml             # Root ArgoCD Application (bootstrap entry point)
│   ├── image-updater-config.yaml    # ArgoCD Image Updater ACR registry config
│   └── applications/                # Child ArgoCD Applications (one file = one app)
│       ├── nginx-ingress.yaml
│       ├── argocd-ingress.yaml
│       ├── prometheus.yaml
│       ├── prometheus-ingress.yaml
│       └── todo-app.yaml
│
└── apps/                            # Kubernetes manifests for each application
    ├── argocd/                      # ArgoCD server patch (--rootpath, --insecure)
    ├── prometheus/                  # Helm values + ingress for monitoring stack
    └── todo-app/                    # Kustomize base/overlay for the todo app
        ├── base/                    # Namespace, Deployment, Service, Ingress
        └── overlays/prod/           # Production overlay
```

## Applications & Access URLs

| Application | URL | Credentials |
|---|---|---|
| Todo App | `http://<LB-IP>/todo` | None |
| ArgoCD | `http://<LB-IP>/argocd` | `admin` / see bootstrap output |
| Grafana | `http://<LB-IP>/grafana` | `admin` / `changeme` |
| Prometheus | `http://<LB-IP>/prometheus` | None |

## How to Deploy a New Application

1. Add a Kubernetes manifest folder under `apps/<app-name>/`
2. Create an ArgoCD Application file at `argocd/applications/<app-name>.yaml`
3. Push to `main` — ArgoCD auto-detects and deploys it

## Bootstrap (Fresh Cluster)

Run the **Bootstrap ArgoCD** GitHub Actions workflow. It will:
1. Install ArgoCD v3.3.6
2. Install ArgoCD Image Updater v0.15.0
3. Create the ACR pull secret
4. Register the GitOps root application

After bootstrap, all apps in `argocd/applications/` are deployed automatically.

## GitOps Behaviour

- `selfHeal: true` — any manual `kubectl` change is reverted within ~3 minutes
- `prune: true` — deleting a file from `argocd/applications/` removes the app from the cluster
- Image Updater polls ACR every 2 minutes and auto-deploys new image digests for `todo-app`

## Required GitHub Secrets & Variables

### Repository Variables (`vars.*`)

| Name | Description |
|---|---|
| `AZURE_CLIENT_ID` | Client ID of the Azure Managed Identity used for OIDC login |
| `AZURE_TENANT_ID` | Azure Active Directory Tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Azure Subscription ID |
| `ACR_USERNAME` | Azure Container Registry admin username |

### Repository Secrets (`secrets.*`)

| Name | Description |
|---|---|
| `ACR_PASSWORD` | Azure Container Registry admin password |

### Where to find these values

- **AZURE_CLIENT_ID / TENANT_ID / SUBSCRIPTION_ID** — Azure Portal → Managed Identity `cloudzen-prod-mi` → Overview
- **ACR_USERNAME / ACR_PASSWORD** — Azure Portal → ACR `cloudzen-prod-acr` → Access keys

