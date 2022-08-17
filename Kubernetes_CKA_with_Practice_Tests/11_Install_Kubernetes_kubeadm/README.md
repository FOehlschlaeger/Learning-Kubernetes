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
