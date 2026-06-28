# MLOps GitOps

Kubernetes manifests and Helm charts for the End-to-End MLOps Platform on Cloud, managed via ArgoCD GitOps.

## Overview

This repository is the **single source of truth** for the Kubernetes cluster state. ArgoCD continuously watches this repository and automatically synchronizes any changes to the cluster.

It follows the **App of Apps** pattern:
- One root ArgoCD application watches this repository
- It automatically creates and manages all child applications (MLflow, model-serving, monitoring)

## Repository Structure

```
mlops-gitops/
├── apps/
│   ├── mlflow.yaml             # ArgoCD Application — MLflow tracking server
│   ├── model-serving.yaml      # ArgoCD Application — FastAPI + Gradio + drift detector
│   └── monitoring.yaml         # ArgoCD Application — Prometheus + Grafana + AlertManager
├── bootstrap/
│   └── app-of-apps.yaml        # Root ArgoCD Application (app-of-apps pattern)
└── charts/
    ├── model-serving/          # Helm chart for model-serving
    │   ├── templates/
    │   │   ├── deployment.yaml         # FastAPI deployment with liveness/readiness probes
    │   │   ├── service.yaml            # ClusterIP service
    │   │   ├── ingress.yaml            # Nginx Ingress (public access)
    │   │   ├── hpa.yaml                # Horizontal Pod Autoscaler (1-3 replicas)
    │   │   └── drift-detector.yaml     # CronJob drift detector (every hour)
    │   ├── Chart.yaml                  # Helm chart metadata
    │   └── values.yaml                 # Default values (image tag, resources, MLflow URI)
    └── grafana-dashboards/
        └── mlops-dashboard.json        # Grafana MLOps dashboard (versioned as code)
```

## Applications

| Application | Namespace | Description |
|---|---|---|
| `mlflow` | `mlflow` | MLflow tracking server + PostgreSQL backend |
| `model-serving` | `model-serving` | FastAPI API + Gradio UI + HPA + drift detector |
| `monitoring` | `monitoring` | Prometheus + Grafana + AlertManager + Slack |

## GitOps Workflow

```
Developer pushes to mlops-model
    ↓
GitHub Actions CI/CD pipeline
    ↓
Updates charts/model-serving/values.yaml
(new Docker image tag)
    ↓
ArgoCD detects change in this repository
    ↓
ArgoCD applies changes to EKS cluster
    ↓
New model version deployed automatically
```

## Key Features

### Auto-healing
ArgoCD is configured with `selfHeal: true` — if any resource is manually deleted or modified in the cluster, ArgoCD automatically restores it to match the state defined in this repository.

### Drift Detection
A Kubernetes CronJob runs every hour to detect data drift in production:
- Compares production data distributions against training baseline
- Sends Slack alert to `#mlops-alerts` if drift detected
- Merges production + training data
- Triggers automated retraining via GitHub Actions

### Monitoring
The Grafana dashboard (`grafana-dashboards/mlops-dashboard.json`) is versioned in this repository, ensuring it persists across infrastructure redeployments.

## ArgoCD Applications

### model-serving
- **Source**: `charts/model-serving/` (Helm chart)
- **Target**: `model-serving` namespace
- **Sync**: Automated with self-healing

### monitoring
- **Source**: `kube-prometheus-stack` Helm chart (v57.0.3)
- **Target**: `monitoring` namespace
- **Alerts**: AlertManager → Slack `#mlops-alerts`

## Secrets Management

Sensitive values (Slack webhook, GitHub token) are **not stored in this repository**. They are managed as Kubernetes Secrets provisioned by Terraform:

```
slack-webhook-url   → kubernetes secret: alertmanager-slack-secret
github-token        → kubernetes secret: github-token
```

## Related Repositories

- [mlops-infrastructure](https://github.com/FatimaGhi/mlops-infrastructure) — AWS infrastructure (Terraform)
- [mlops-model](https://github.com/FatimaGhi/mlops-model) — ML model code and API