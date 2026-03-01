### Create local VM
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
