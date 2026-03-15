### Add Monitoring (Prometheus + Grafana)

This section describes how to deploy a Kubernetes monitoring stack using Prometheus and Grafana with the kube-prometheus-stack Helm chart.

The stack collects:
- Node metrics (CPU, memory, disk)
- Pod metrics
- Kubernetes object metrics
- Cluster metrics
- Alerting rules

# Architecture

Kubernetes Cluster
        │
        │
+----------------------+
|   Node Exporter      |
|   Kube State Metrics |
|   Kubelet Metrics    |
+----------------------+
        │
        ▼
   Prometheus
        │
        ▼
     Grafana
        │
        ▼
  Dashboards & Alerts

# Repository Structure

platform-environments
│
├── helm
│   └── monitoring
│       ├── values.yaml
│       └── README.md
│
└── envs
    └── dev
        └── monitoring.yaml

# Step 1 — Create monitoring values file
Create:
```bash
helm/monitoring/values.yaml
```

```YAML
grafana:
  adminPassword: admin123
  service:
    type: ClusterIP

prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false

alertmanager:
  enabled: true

nodeExporter:
  enabled: true

# Disable control plane metrics (useful for EKS or managed clusters)

kubeEtcd:
  enabled: false

kubeControllerManager:
  enabled: false

kubeScheduler:
  enabled: false

coreDns:
  enabled: false
```

This configuration:
- enables Grafana
- enables Prometheus
- enables Alertmanager
- enables Node Exporter
- disables control-plane metrics (for managed clusters like EKS)

# Step 2 — Create ArgoCD Application manifest

Create:
```bash
envs/dev/monitoring.yaml
```

```YAML
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/techngi/platform-environments
    targetRevision: main
    path: helm/monitoring

  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

This tells ArgoCD to:
- watch the repository
- deploy the monitoring Helm chart
- automatically sync changes

# Step 3 — Add Helm repository

Before installing the monitoring stack, add the Helm repository.
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

# Step 4 — Install monitoring stack

Deploy the stack using Helm.

```bash
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --create-namespace \
  -f helm/monitoring/values.yaml
```

This installs:
- Prometheus
- Grafana
- Alertmanager
- Node Exporter
- Kube State Metrics
- Prometheus Operator

# Step 5 — Verify installation

Check the pods:

```bash
kubectl get pods -n monitoring
```
Expected components:

```bash
monitoring-grafana
monitoring-kube-prometheus-prometheus
monitoring-kube-prometheus-alertmanager
monitoring-kube-state-metrics
monitoring-prometheus-node-exporter
```

Check services:

```bash
kubectl get svc -n monitoring
```

Check Helm release:

```bash
helm list -n monitoring
```

# Step 6 — Access Grafana

Forward the Grafana service port:

```bash
kubectl port-forward --address 0.0.0.0 svc/monitoring-grafana -n monitoring 3000:80
```

Open Grafana in your browser:

```bash
http://<VM-IP>:3000
```

Login credentials:

```bash
username: admin
password: admin123
```

# Step 7 — Access Prometheus

Forward the Prometheus service:

```bash
kubectl port-forward --address 0.0.0.0 svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090
```

Open Prometheus:

```
http://<VM-IP>:9090
```

# Step 8 — Verify Prometheus targets

Open:

```bash
Status → Targets
```

You should see targets such as:

```bash
node-exporter
kube-state-metrics
apiserver
grafana
alertmanager
```

All targets should show:

```bash
State = UP
```

# Step 9 — Explore Grafana dashboards

The Helm chart automatically installs dashboards.

Navigate to:

```bash
Dashboards → Kubernetes
```

Example dashboards:

```bash
Kubernetes / Compute Resources / Cluster
Kubernetes / Compute Resources / Node
Kubernetes / Compute Resources / Namespace (Pods)
Kubernetes / Networking / Cluster
```

These dashboards display:
- node CPU usage
- node memory usage
- pod CPU usage
- pod memory usage
- network traffic

# Step 10 — Generate test metrics

Deploy a sample workload to generate metrics.

```bash
kubectl create deployment nginx-demo --image=nginx
kubectl scale deployment nginx-demo --replicas=5
kubectl expose deployment nginx-demo --port=80
```

Refresh Grafana dashboards to observe:
- pod metrics
- CPU usage
- memory usage

# Step 11 — Generate CPU load (optional)

Create a stress test pod:

```bash
kubectl run cpu-test \
--image=busybox \
--restart=Never \
-- /bin/sh -c "while true; do :; done"
```

Now check Grafana dashboards to observe CPU spikes.

Metrics Collected

The monitoring stack collects:
# Node metrics
Provided by Node Exporter

Examples:

```bash
- CPU usage
- Memory usage
- Filesystem usage
- Network traffic
- Load average
- Kubernetes object metrics

Provided by kube-state-metrics

Examples:

```bash
- pod status
- deployment replicas
- container restarts
- job status
- Kubernetes API metrics

Collected from the API server

Examples:

```bash
API request latency
API error rate
API request volume
Alerting
```

Alertmanager processes alerts generated by Prometheus rules.

Example alerts:

```bash
Node CPU usage high
Node memory usage high
Pod crash loops
Kubernetes API latency
```

Alerts can be configured to send notifications via:

```bash
- Slack
- Email
- PagerDuty
- Webhook
- Result
```

You now have a production-grade Kubernetes monitoring stack consisting of:

```bash
Prometheus  → metrics collection
Grafana     → dashboards and visualization
Alertmanager → alert routing
Node Exporter → node metrics
Kube State Metrics → Kubernetes object metrics
```
