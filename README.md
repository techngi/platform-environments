# 🌍 Platform Environments (GitOps Deployment)

This repository manages Kubernetes deployments using **GitOps principles** with ArgoCD and Helm.

---

## 🧩 Overview

This repo represents the **desired state of infrastructure and applications**, enabling automated and reliable deployments.

It supports:

* Multi-environment deployments (Dev & Prod)
* Helm-based configuration
* ArgoCD continuous reconciliation

---

## 🏗 Architecture

```
AWS EKS Cluster
│
├── Namespace: dev
│   └── platform-app (dev config)
│
└── Namespace: prod
    └── platform-app (prod config)
```

---

## 🔁 GitOps Workflow

```
CI Pipeline (Jenkins)
        ↓
Update values.yaml (image tag)
        ↓
Git Push (this repo)
        ↓
ArgoCD detects change
        ↓
Sync → Deploy to Kubernetes
```

---

## 🌱 Environments

### 🔹 Dev

* Automatic deployment
* Used for testing changes
* Lower resource configuration

### 🔹 Prod

* Manual promotion from Dev
* Higher availability configuration
* Controlled via approval gate

---

## 📂 Repository Structure

```
.
├── helm/
│   └── platform-app/
│
├── envs/
│   ├── dev/
│   │   └── values.yaml
│   └── prod/
│       └── values.yaml
│
└── apps/
    ├── dev/
    │   └── platform-app.yaml
    └── prod/
        └── platform-app.yaml
```

---

## ⚙️ Helm Configuration

* Uses shared Helm chart
* Environment-specific values via `values.yaml`
* ConfigMaps used for:

  * APP_VERSION
  * APP_ENV
  * LOG_LEVEL

---

## 📊 Monitoring & Observability

* Prometheus integrated for metrics collection
* Grafana dashboards for visualization
* Application exposes metrics endpoint

---

## 🛡 Reliability Features

* Liveness & Readiness probes
* PodDisruptionBudget
* Horizontal Pod Autoscaler (HPA)
* Self-healing via ArgoCD

---

## 🔐 Security

* Images scanned via Trivy before deployment
* Only verified images promoted to production

---

## 🚀 Deployment Strategy

* Git-driven deployments (GitOps)
* Immutable image versions (Git SHA)
* Automated sync via ArgoCD

---

## 🎯 Key Benefits

* Declarative infrastructure
* Environment consistency
* Easy rollback using Git
* Reduced manual intervention

---
