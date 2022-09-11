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
  - install `kubeadm` tool with `kubelet` and `kubectl` on all nodes
  - initialize master server to install and configure all required components on master server 
  - **before joining worker nodes: ensure network prerequisites are met for the Kubernetes Pod Network**
  - join worker nodes to master node to form a kubernetes cluster

---
## Provision VMs using Vagrant
- install Virtualbox on host machine
- **for nested VirtualBox usage: in VirtualBox on origin host, activate option `Nested VT-x/AMD-V aktivieren`**
```
sudo apt update
sudo apt install virtualbox
```
- use CLI tool [Vagrant](https://www.vagrantup.com/) and [install it on linux host](https://www.vagrantup.com/downloads), where the virtual Kubernetes cluster should be set up
- clone repo [KodeKloud: certified-kubernetes-administrator-course](https://github.com/kodekloudhub/certified-kubernetes-administrator-course) to use `Vagrantfile`
- modified [Vagrantfile](./Vagrantfile) to provision VMs using vagrant CLI tool
- set up and provision VMs using vagrant, first master, then both worker nodes
```
vagrant up
```
- check status of VMs
```
vagrant status
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

---
## Initialize controlplane using `kubeadm` and arguments
```
kubeadm init --apiserver-advertise-address=10.35.215.6 --pod-network-cidr=10.244.0.0/16 --apiserver-cert-extra-sans=controlplane
```

## Setup default .kube/config file
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Join other nodes to cluster
Create new token to join using on controlplane node
```
kubeadm token create --print-join-command
```

or from output of `kubeadm init` command
```
kubeadm join 10.35.215.6:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

## Install CNI ([flannel](https://github.com/flannel-io/flannel))
```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```
