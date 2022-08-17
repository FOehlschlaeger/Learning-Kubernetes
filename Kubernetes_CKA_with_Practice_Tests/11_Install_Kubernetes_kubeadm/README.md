# Install Kubernetes using `kubeadm`

[Installing a Kubernetes Cluster using the tool `kubeadm`](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

## References
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- https://github.com/kodekloudhub/certified-kubernetes-administrator-course


---
## Setup `kubeadm`
- tool to automatically take care of installing kubernetes resources (kube-apiserver, etcd, scheduler, controller-manager), setup kubelet on worker nodes, create certificates etc.
- high-level steps:
  - have multiple VMs/servers for configuring a cluster and designate master and worker nodes across machines
  - install container runtime (such as `containerd`, `docker`) on each host machine
  - install `kubeadm` tool on all nodes
  - initialize master server to install and configure all required components on master server 
  - **before joining worker nodes: ensure network prerequisites are met for the Kubernetes Pod Network**
  - join worker nodes to master node to form a kubernetes cluster


## Provision VMs using Vagrant
- install Virtualbox on host machine
- **for nested VirtualBox usage: in VirtualBox on origin host, activate option `Nested VT-x/AMD-V aktivieren`**
```
sudo apt update
sudo apt install virtualbox
```
- use tool [Vagrant](https://www.vagrantup.com/) and [install it on linux host](https://www.vagrantup.com/downloads), where the virtual Kubernetes cluster should be set up
- from repo [KodeKloud: certified-kubernetes-administrator-course](https://github.com/kodekloudhub/certified-kubernetes-administrator-course) use `Vagrantfile`
- clone repo
- updated `Vagrantfile`
```
# -*- mode: ruby -*-
# vi:set ft=ruby sw=2 ts=2 sts=2:

# Define the number of master and worker nodes
# If this number is changed, remember to update setup-hosts.sh script with the new hosts IP details in /etc/hosts of each VM.
NUM_MASTER_NODE = 1
NUM_WORKER_NODE = 2

IP_NW = "192.168.56."
MASTER_IP_START = 1
NODE_IP_START = 2

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  # config.vm.box = "base"
  config.vm.box = "ubuntu/bionic64"

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  config.vm.box_check_update = false
  
  # timeout value
  config.vm.boot_timeout = 600

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  # config.vm.provider "virtualbox" do |vb|
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end
  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Provision Master Nodes
  (1..NUM_MASTER_NODE).each do |i|
      config.vm.define "kubemaster" do |node|
        # Name shown in the GUI
        node.vm.provider "virtualbox" do |vb|
            vb.name = "kubemaster"
            vb.memory = 4096
            vb.cpus = 2
        end
        node.vm.hostname = "kubemaster"
        node.vm.network :private_network, ip: IP_NW + "#{MASTER_IP_START + i}"
        node.vm.network "forwarded_port", guest: 22, host: "#{2710 + i}"

        node.vm.provision "setup-hosts", :type => "shell", :path => "ubuntu/vagrant/setup-hosts.sh" do |s|
          s.args = ["enp0s8"]
        end

        node.vm.provision "setup-dns", type: "shell", :path => "ubuntu/update-dns.sh"

      end
  end


  # Provision Worker Nodes
  (1..NUM_WORKER_NODE).each do |i|
    config.vm.define "kubenode0#{i}" do |node|
        node.vm.provider "virtualbox" do |vb|
            vb.name = "kubenode0#{i}"
            vb.memory = 2048
            vb.cpus = 2
        end
        node.vm.hostname = "kubenode0#{i}"
        node.vm.network :private_network, ip: IP_NW + "#{NODE_IP_START + i}"
                node.vm.network "forwarded_port", guest: 22, host: "#{2720 + i}"

        node.vm.provision "setup-hosts", :type => "shell", :path => "ubuntu/vagrant/setup-hosts.sh" do |s|
          s.args = ["enp0s8"]
        end

        node.vm.provision "setup-dns", type: "shell", :path => "ubuntu/update-dns.sh"
    end
  end
end
```
- check status of VMs
```
vagrant status
```
- set up and provision VMs using vagrant, first master, then both worker nodes
```
vagrant up
```
- ssh into one machine, i.e. `kubemaster`
```
vagrant ssh kubemaster
```
- stop VMs
```
vagrant halt
```
- update single VM from config `Vagrantfile`, i.e. `kubenode01`
```
vagrant reload kubenode01
```
- delete single VM, i.e. `kubenode01`
```
vagrant destroy kubenode01
```
