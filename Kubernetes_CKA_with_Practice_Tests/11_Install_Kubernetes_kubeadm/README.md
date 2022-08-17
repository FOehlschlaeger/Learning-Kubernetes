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
```
sudo apt update
sudo apt install virtualbox
```
- use tool [Vagrant](https://www.vagrantup.com/) and [install it on linux host](https://www.vagrantup.com/downloads), where the virtual Kubernetes cluster should be set up
- from repo [KodeKloud: certified-kubernetes-administrator-course](https://github.com/kodekloudhub/certified-kubernetes-administrator-course) use `Vagrantfile`
- clone repo
- in `Vagrantfile`
```
...
NUM_MASTER_NODE = 1 # number of master nodes to provision
NUM_WORKER_NODE = 2 # number of worker nodes to provision

IP_NW = "192.168.56." # network range for VMs
...
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
- **for nested VirtualBox usage: in VirtualBox on origin host, activate option `Nested VT-x/AMD-V aktivieren`**
