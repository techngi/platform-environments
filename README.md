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
