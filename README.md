# 📦 platform-environments

GitOps deployment repository for the Platform Application.  
This repository manages Kubernetes deployments on AWS EKS using **Helm + ArgoCD**.

---

## 🧠 Overview

This repository is responsible for:

- Deploying the application to EKS
- Managing environment-specific configuration (dev / prod)
- Enforcing production reliability patterns
- Handling GitOps-based rollbacks
- Implementing autoscaling and availability controls

The application source and CI pipeline live in the `platform-app` repository.  
This repo represents the **declarative desired state** of the cluster.

---

## 🏗 Architecture

```
GitHub (platform-app)
        │
        ▼
     Jenkins
        │
        ▼
       ECR
        │
        ▼
     ArgoCD
        │
        ▼
        EKS
   (dev / prod)
```

Flow:

1. Code pushed to GitHub
2. Jenkins builds Docker image
3. Image pushed to ECR
4. GitOps values updated
5. ArgoCD syncs desired state to EKS

---

## 📂 Repository Structure

```
envs/
  ├── dev/
  └── prod/

helm/platform-app/
  ├── templates/
  ├── Chart.yaml
  └── values.yaml
```

- `envs/dev` → Development overrides
- `envs/prod` → Production overrides
- `helm/platform-app` → Core Helm chart

---

# 🚀 Production Hardening & Reliability

This deployment implements production-grade controls.

---

## 1️⃣ Health Management

### Liveness & Readiness Probes

Configured in `deployment.yaml` via Helm values.

- Liveness → Restarts unhealthy containers
- Readiness → Prevents traffic to unhealthy pods
- `/health` endpoint used for validation

Improves availability and zero-downtime deployments.

---

## 2️⃣ Resource Management

CPU and memory requests/limits defined in `values.yaml`.

Benefits:

- Prevents noisy neighbor issues
- Required for effective HPA
- Enables predictable scaling behavior

---

## 3️⃣ Horizontal Pod Autoscaler (HPA)

Configured using `autoscaling/v2`.

- CPU Target: 60%
- Memory Target: 70%
- Min Replicas: 2
- Max Replicas: 8
- Custom scale-up / scale-down behavior

Validation commands:

```
kubectl get hpa -n devops-platforms
kubectl top pods -n devops-platforms
```

Load testing supported via temporary busybox pod.

---

## 4️⃣ PodDisruptionBudget (PDB)

Ensures availability during:

- Node drain
- Cluster upgrades
- Rolling maintenance

Configured with:

- `maxUnavailable: 1`

Prevents service downtime during voluntary disruptions.

---

## 🔁 GitOps Rollback Strategy

### Primary Rollback (Recommended)

```
git revert <bad-commit>
git push
```

ArgoCD automatically syncs cluster state.

Benefits:

- Full audit trail
- Immutable history
- Clean GitOps model

---

### Emergency Rollback (ArgoCD CLI)

```
argocd app history <app-name>
argocd app rollback <app-name> <revision>
```

---

## 🛠 Operational Commands

Check application status:

```
kubectl get pods -n devops-platforms
kubectl describe deploy <deployment>
kubectl logs <pod>
```

Check Argo sync:

```
argocd app get <app-name>
```

---

## 🎯 Key Platform Capabilities Demonstrated

- GitOps-based deployment
- Helm templating
- Autoscaling with tuned behavior
- Health management
- Resource governance
- High availability controls
- Declarative rollback strategy

---

## 📌 Related Repository

Application source and CI pipeline:

👉 https://github.com/techngi/platform-apps
