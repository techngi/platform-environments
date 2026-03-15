# Fluent Bit Logging for Amazon EKS

## Overview

This component deploys **Fluent Bit as a DaemonSet** in an Amazon EKS cluster to collect container logs and ship them to **AWS CloudWatch Logs**.

Fluent Bit runs on every node and tails container log files from Kubernetes, enriches them with Kubernetes metadata, and forwards them to a centralized log group in CloudWatch.

This enables:

- Centralized logging across the entire Kubernetes cluster
- Faster troubleshooting and incident investigation
- Log search and analytics using CloudWatch Logs Insights
- Alerting based on log patterns

---

## Architecture

```
Kubernetes Pods
   │
   ▼
Container Logs (/var/log/containers)
   │
   ▼
Fluent Bit DaemonSet
   │
   ▼
AWS CloudWatch Logs
   │
   ▼
CloudWatch Logs Insights / Alerts
```

Each node runs a Fluent Bit pod that collects logs from containers and sends them to CloudWatch using **IAM Roles for Service Accounts (IRSA)**.

---

# Prerequisites

Before deploying Fluent Bit ensure the following:

- Amazon EKS cluster is running
- `kubectl` is configured
- AWS CLI configured
- Helm installed
- OIDC provider enabled on the cluster

Check OIDC:

```bash
aws eks describe-cluster \
  --name devops-hybrid \
  --region ap-southeast-2 \
  --query "cluster.identity.oidc.issuer"
```

# Deployment Steps

1. Create IAM Policy for CloudWatch Logs

Create a policy allowing Fluent Bit to write logs to CloudWatch.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

Create the policy:

```bash
aws iam create-policy \
  --policy-name FluentBitCloudWatchLogsPolicy \
  --policy-document file://fluentbit-cloudwatch-policy.json
```

2. Create IAM Role for Service Account (IRSA)

```bash
eksctl create iamserviceaccount \
  --cluster devops-hybrid \
  --namespace logging \
  --name aws-for-fluent-bit \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/FluentBitCloudWatchLogsPolicy \
  --approve \
  --region ap-southeast-2
```

Verify the service account:

```bash
kubectl get sa aws-for-fluent-bit -n logging -o yaml
```

You should see:

```bash
eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/...
```

3. Install Fluent Bit using Helm

Create namespace and Install Fluent Bit:

```bash
helm upgrade --install aws-for-fluent-bit eks/aws-for-fluent-bit \
  --namespace logging \
  --create-namespace \
  -f helm/fluent-bit/values.yaml
```

## Verification

Check Fluent Bit pods:

```bash
kubectl get pods -n logging
```

Check Fluent Bit logs:

```bash
kubectl logs -n logging -l app.kubernetes.io/name=aws-for-fluent-bit
```

Verify CloudWatch log group:

```bash
aws logs describe-log-groups \
  --region ap-southeast-2 \
  --log-group-name-prefix "/aws/eks/fluentbit-cloudwatch"
```

Check log streams:

```bash
aws logs describe-log-streams \
  --log-group-name /aws/eks/fluentbit-cloudwatch/logs \
  --region ap-southeast-2
```

Example query:

```sql
fields @timestamp, log
| filter log like /ERROR/
| sort @timestamp desc
| limit 50
```

Example workflow:

```
Fluent Bit
  │
  ▼
CloudWatch Logs
  │
  ▼
Metric Filter
  │
  ▼
CloudWatch Alarm
  │
  ▼
SNS Notification
```

Example alert pattern:
```
ERROR
```
This can trigger alerts via:
- Email
- Slack
- PagerDuty
- OpsGenie

# Troubleshooting
- IRSA not working

Check service account annotation:
```bash
kubectl get sa aws-for-fluent-bit -n logging -o yaml
```

- Wrong AWS region

Ensure Fluent Bit output configuration matches the cluster region:
```bash
region ap-southeast-2
```

- CloudFormation stack failure
Delete failed stack and recreate IRSA:
```bash
aws cloudformation delete-stack ...
```

- Fluent Bit not sending logs
Check Fluent Bit logs:
```bash
kubectl logs -n logging <fluent-bit-pod>
```

## Key Benefits

- Centralized cluster logging
- Persistent logs even if pods restart
- Faster incident debugging
- Log analytics using CloudWatch Logs Insights
- Alerting based on log patterns
