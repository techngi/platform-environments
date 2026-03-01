# Create EKS in the cheapest way (NO NAT Gateway)

- Download and install eksctl

```bash
curl -s --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

- Create and configure AWS credentials
* Create AWS IAM user
** Configure the user

```bash
aws configure
# set region: ap-southeast-2 (Sydney)
aws sts get-caller-identity
```

- Create AWS Cluster

* Create YAML file to setup cluster in AWS

```bash
cat > cluster-public.yaml <<'YAML'
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: devops-hybrid
  region: ap-southeast-2
  version: "1.32"

vpc:
  nat:
    gateway: Disable

managedNodeGroups:
  - name: ng-dev
    instanceType: t3.small
    desiredCapacity: 2
    minSize: 1
    maxSize: 2
    privateNetworking: false
    volumeSize: 30
    ssh:
      allow: false

iam:
  withOIDC: true
YAML
```

```bash
eksctl create cluster -f cluster-public.yaml

aws eks describe-cluster --name devops-hybrid --region ap-southeast-2

aws eks update-kubeconfig --name devops-hybrid --region ap-southeast-2

kubectl get pods -A
```

# Create ArgoCD docker in AWS Cluster and access it

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl -n argocd get pods

kubectl -n argocd port-forward svc/argocd-server 8085:80

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

On Host from vagrant folder
vagrant ssh-config
ssh -i [IdentityFile] -L 8085:localhost:8085 vagrant@192.168.56.10

- Now you can access from host machine
http://localhost:8085
```

# Create Helm Chart

```bash
mkdir -p helm
helm create helm/platform-app
ls -la helm/platform-app
```

It should include Chart.yaml, values.yaml, templates/, etc.

platform-environments/
├── helm/
│   └── platform-app/
│       ├── Chart.yaml
│       ├── templates/
│       └── values.yaml        ← default values
│
envs/
    ├── dev/
    │   └── values.yaml        ← dev overrides
    └── prod/
        └── values.yaml        ← prod overrides
