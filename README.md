### Overview - GitOps deployment repository for Platform App (ArgoCD + Helm + EKS)
This repository contains GitOps deployment configuration for the Platform App. It manages application deployment to AWS EKS using ArgoCD + Helm. Values for dev and prod environments are stored separately.

### Hybrid Setup using local VM and AWS EKS

- Create local VMs

VM1: jenkins-ci
Jenkins (Docker preferred)
Docker engine
Trivy scanner
Git + tools

Hardware Specs - 4 vCPU, 8GB RAM, 60GB disk

```bash
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  # Base box
  config.vm.box = "ubuntu/jammy64"

  # Resize primary disk to 60GB
  config.disksize.size = "60GB"

  config.vm.define "jenkins-ci" do |jc|

    jc.vm.hostname = "jenkins-ci"

    # Networking
    jc.vm.network "private_network", ip: "192.168.56.10"

    # VirtualBox provider settings
    jc.vm.provider "virtualbox" do |vb|
      vb.name   = "jenkins-ci"
      vb.memory = 8192    # 8GB RAM
      vb.cpus   = 4       # 4 CPU cores
    end

  end

end
```

- Create GitOps repo on GitHub

```bash
mkdir -p ~/platform-environments
cd ~/platform-environments
git init
sudo nano README.md
git add README.md
git config --global user.email "sanaqvi573@gmail.com"
git config --global user.name "Syed Sohail Abbas"
git commit -m "Initial Commit"
git remote add origin https://github.com/techngi/platform-environments.git
git push -u origin master

mkdir -p helm/week3-app envs/dev envs/prod
```

- Install utilities

```bash
sudo apt update && sudo apt -y upgrade
sudo apt -y install ca-certificates curl gnupg lsb-release git unzip jq
```

- Install Docker

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
docker version
```

- Run Jenkins in Docker + persistent storage

```bash
sudo mkdir -p /srv/jenkins_home # Create folders
sudo chown -R 1000:1000 /srv/jenkins_home
```

- Run Jenkins:

```bash
DOCKER_GID=$(getent group docker | cut -d: -f3)

docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --group-add ${DOCKER_GID} \
  jenkins/jenkins:lts
```

Install Docker  and AWS CLI inside Jenkins

```bash
docker exec -u root -it jenkins bash -lc '
apt-get update &&
apt-get install -y docker.io &&
docker --version'
```

```bash
docker exec -u root -it jenkins bash -lc '
apt-get update &&
apt-get install -y unzip curl &&
curl -sS "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip &&
unzip -q /tmp/awscliv2.zip -d /tmp &&
/tmp/aws/install &&
aws --version
```


- Get initial admin password:

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

- Open Jenkins from host:
http://192.168.56.10:8080

- Install plugins (minimum):
* Pipeline
* Git
* Credentials Binding
* Docker Pipeline
* Blue Ocean (optional)
* GitHub integration (optional)

- Install CI tools on VM1 (AWS + K8s CLIs + scanners)

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

- Install KubeCtl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

- Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

- Install Trivy (image scan)

```bash
sudo apt -y install wget
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo gpg --dearmor -o /etc/apt/keyrings/trivy.gpg
echo "deb [signed-by=/etc/apt/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt update && sudo apt -y install trivy
trivy -v
```

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

# Create ECR Repository and upload image

```bash
aws ecr create-repository --repository-name devops-app --region ap-southeast-2
aws ecr get-login-password --region ap-southeast-2 | docker login --username AWS --password-stdin 421869852482.dkr.ecr.ap-southeast-2.amazonaws.com


docker pull sanaqvi573/week3-app:latest
docker tag sanaqvi573/week3-app:latest 421869852482.dkr.ecr.ap-southeast-2.amazonaws.com/devops-app:latest
docker push 421869852482.dkr.ecr.ap-southeast-2.amazonaws.com/devops-app:latest
aws ecr list-images --repository-name devops-app --region ap-southeast-2
```

# Update values.yaml and deployment.yaml files

- envs/dev/values.yaml
  
```bash
replicaCount: 1
image:
  repository: 421869852482.dkr.ecr.ap-southeast-2.amazonaws.com/devops-app
  tag: "latest"
  pullPolicy: IfNotPresent
```

- helm/platform-app/templates/deployment.yaml
  
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "platform-app.fullname" . }}
  labels:
    {{- include "platform-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "platform-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "platform-app.labels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "platform-app.serviceAccountName" . }}
      {{- with .Values.podSecurityContext }}
      securityContext:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Chart.Name }}
          {{- with .Values.securityContext }}
          securityContext:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.containerPort }}
              protocol: TCP
          {{- with .Values.livenessProbe }}
          livenessProbe:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.readinessProbe }}
          readinessProbe:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.volumeMounts }}
          volumeMounts:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- with .Values.volumes }}
      volumes:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

# Create ArgoCD App

- Install ArgoCD CLI

```bash
sudo curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo chmod +x /usr/local/bin/argocd
argocd version --client

argocd login localhost:8085 --username admin --password <PASTE_PASSWORD> --insecure

kubectl create namespace devops-platforms
```bash

- Create the app

```bash
argocd app create platforms-app-dev --repo https://github.com/techngi/platform-environments.git --path helm/platform-app --dest-namespace devops-platforms --dest-server https://kubernetes.default.svc --values ../../envs/dev/values.yaml --sync-policy automated
```

### Production Hardening & Reliability Enhancements
1 - Liveness & Readiness Probes
Liveness probe ensures unhealthy containers are restarted automatically.
Readiness probe ensures traffic is only routed to healthy pods.
Improves availability and prevents failed deployments from serving traffic.

* Helm Template Support (deployment.yaml)

```bash
{{- with .Values.livenessProbe }}
livenessProbe:
  {{- toYaml . | nindent 12 }}
{{- end }}
{{- with .Values.readinessProbe }}
readinessProbe:
  {{- toYaml . | nindent 12 }}
{{- end }}
```

values.yaml Configuration

```bash
livenessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 15
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
```

2 - Resource Requests & Limits

deployment.yaml Verification

```bash
resources:
  {{- toYaml . | nindent 12 }}
```

values.yaml Configuration
Guarantees minimum compute allocation.
Prevents noisy neighbor issues.
Required for effective HPA autoscaling.

- Aligns with Kubernetes best practices for production workloads.

```bash
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```bash

3️⃣ Horizontal Pod Autoscaler (HPA)
Autoscaling is configured using Kubernetes autoscaling/v2 API with CPU and memory metrics.

Enables dynamic scaling based on workload.
Prevents over-provisioning.
Demonstrates production-grade elasticity.

- Metrics Server Validation

```bash
kubectl get deployment metrics-server -n kube-system
kubectl top pods -n <namespace>
kubectl top nodes
```

HPA Configuration (hpa.yaml)

```bash
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: platform-app-hpa
  namespace: devops-platforms
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: platform-app
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 180
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
```

Helm Integration

deployment.yaml:

```bash
{{- if not .Values.autoscaling.enabled }}
replicas: {{ .Values.replicaCount }}
{{- end }}
```

values.yaml:

```bash
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 8
  targetCPUUtilizationPercentage: 60
  targetMemoryUtilizationPercentage: 70
```

Load Testing to Validate Scaling

```bash
kubectl run -i --tty loadgen --rm --image=busybox -- /bin/sh

Inside pod:

while true; do wget -q -O- http://platforms-app-dev-platform-app.devops-platforms.svc.cluster.local/health >/dev/null; done

Monitor:

kubectl get hpa -n devops-platforms -w
kubectl get deploy platforms-app-dev-platform-app -n devops-platforms -w
kubectl top pods -n devops-platforms --watch

```

5. PodDisruptionBudget (PDB)

PDB protects availability during voluntary disruptions such as:

Node drain

Cluster upgrades

Rolling maintenance

It does not protect against application crashes.

templates/pdb.yaml

```bash
{{- if .Values.pdb.enabled }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "platform-app.fullname" . }}-pdb
  labels:
    {{- include "platform-app.labels" . | nindent 4 }}
spec:
  maxUnavailable: {{ .Values.pdb.maxUnavailable }}
  selector:
    matchLabels:
      {{- include "platform-app.selectorLabels" . | nindent 6 }}
{{- end }}
```


values.yaml:

``` bash
pdb:
  enabled: true
  maxUnavailable: 1

Verify:

kubectl get pdb -n devops-platforms
kubectl describe pdb <release-name>-pdb -n devops-platforms
```


### GitOps-Based Rollback Strategy

This project follows GitOps best practices with ArgoCD.

ArgoCD automatically syncs and restores the previous stable state.
This ensures:
Full audit trail
Version-controlled recovery
Immutable infrastructure history


```bash
Primary Rollback — Git Revert
git revert <bad-commit-sha>
git push
```

Emergency Rollback — ArgoCD CLI

```bash
kubectl -n argocd port-forward svc/argocd-server 8081:443

kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d && echo

argocd login localhost:8081 --insecure

argocd app history <app-name>
argocd app rollback <app-name> <history-id>
```bash

Application Rollback Flow

```bash
git log
git revert <bad-commit>
git push

```
ArgoCD detects drift and restores cluster state automatically.

