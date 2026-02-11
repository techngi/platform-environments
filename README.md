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
