# Step 1 — Create monitoring values file
Create:
helm/monitoring/values.yaml

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
