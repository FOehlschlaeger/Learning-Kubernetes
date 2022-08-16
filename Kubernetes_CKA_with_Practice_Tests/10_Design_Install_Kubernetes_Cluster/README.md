# Design and Install a Kubernetes Cluster

## References
- [Youtube Channel: Setup Kubernetes the Hard Way](https://www.youtube.com/watch?v=uUupRagM7m0&list=PL2We04F3Y_41jYdadX55fdJplDvgNGENo)
- [Github Repo: Setup Kubernetes the Hard Way](https://github.com/mmumshad/kubernetes-the-hard-way)

---
## Cluster Design in Udemy Course
- Local virtual cluster on one machine
- Virtualbox
- CNI: Weave Network
- 1 Loadbalancer
- 3 Master Nodes with stacked topology (ETCD on master nodes)
- 2 Worker Nodes

## Design a Kubernetes Cluster
- purpose
  - education: `minikube` or single node cluster with `kubeadm`
- development and testing
  - multi-node cluster with single master and multiple worker nodes
  - setup with `kubeadm` tool
- hosting prodcution applications
  - highly available multi node cluster with multiple master nodes
  - `kubeadm` or KOPS on AWS or GCP
  - upto 5000 nodes
  - upto 150k pods in cluster
  - upto 300k total containers
  - upto 100 pod per node
- use `kubeadm` for on-prem hosting 
- storage
  - high performance: SSd backed storage
  - multiple concurrent connections: network based storage
  - persistent shared volumes for shared access across multiple pods
  - label nodes with specific disk types
  - use node selectors to assign applications to nodes with specific disk types
- nodes
  - virtual or physical machines
  - minimum of 4 nodes
  - master vs worker nodes
  - linux x86_64 architecture
  - best practice: do not host workloads on master nodes (`kubeadm` adds taint to master nodes to prevent hosting workloads on master nodes)
  - in large cluster: separate ETCD on extra servers, not stacked cluster with ETCD on master nodes

## Kubernetes Infrastructure
- setup on linux with binaries or use linux VMs on Windows since Docker images are linux-based
- Kubernetes on local machine
  - `minikube`: deploys VM, only single node cluster
  - `kubeadm` tool: requires VM to be ready, for single/multi node cluster
- turnkey solutions self-managed in terms of
  - provisioning and configuration of VMs
  - using scripts to deploy cluster
  - maintaining VMs
  - `Openshift`: on-prem platform by Redhat
  - `Cloud Foundry Container Runtime`: open source
  - `VMWare Cloud PKS`
  - `Vagrant`
- hosted solutions as Kubernetes-as-a-Service by Cloud Provider (AWS, GKE)

### Chosen Setup
- local setup using several VMs on VirtualBox

## Configuration of High Availability
- multiple instances of master and control plane components
- master node usually with controle plane components: ETCD, API Server, Controller Manager, Scheduler
- `kubectl` utility points to master on `https://master1:6443` (configured in `~/.kube/config` file, to reach kube-API Server)
- load balancer `https://load-balancer:6443` in front of multiple master nodes to split traffic/requests between api servers `https://master1:6443` and `https://master2:6443`
  - point `kubectl` to load balancer which forwards traffic to one of kube-api servers (master nodes)
  - `Nginx` or `HAproxy` as load balancer
- `controller-manager` and `scheduler` watch state of cluster and pods (recreation via `ReplicaSets` etc.)
  - one instance is `active`, others are `standby` due to leader-election process
  - setup instance of `kube-controller-manager --leader-elect true --leader-elect-lease-duration 15s --leader-elect-renew-deadline 10s --leader-elect-retry-period 2s` (default)
  - the instance updating kube-controller-manager endpoint first becomes leader instance
  - logs of state are written and acquired when one instance is unavailable
- ETCD 
  - stacked topology: ETCD is part of master node with other Kubernetes components
  - external ETCD topology: ETCD on external servers for higher secure environment
  - kube-api server needs to point to ETCD servers
  - `cat /etc/systemd/system/kube-apiserver.service`
```
...
  --etcd-server=https://ETCD-SERVER-IP1:2379,https://ETCD-SERVER-IP2:2379
...
```

## ETCD in HA (High-Availability)
- ETCD: distributed reliable key-value store that is simple, secure and fast
- ETCD distributed on several different nodes in cluster, each of these nodes having the same data in ETCD
- data consistency between ETCD instances due to election process with one single leader and remaining follower nodes
- request of change in data (write) send to follower node, is fowarded internally to leader node and leader node write copy of change to all follower nodes
- write process is considered to be complete, if quorum of follower nodes send back their consent to leader node
- `quorum = N/2 + 1 with N: number of servers`: minimum number of nodes in cluster to function properly
- 3 or 5 master nodes is recommended to increase fault tolerance of cluster
- best practice: **odd number of master nodes / odd number of ETCD instances in cluster**
- even number can lead to non-working cluster due to network partition problems because of no quorum

| Instances | Quorum | Fault Tolerance |
| --- | --- | --- |
| 1 | 1 | 0 |
| 2 | 2 | 0 |
| 3 | 2 | 1 |
| 4 | 3 | 1 |
| 5 | 3 | 2 |

### Election Process
- RAFT
- random timer on each instance, first instance is elected leader
- if leader node is unreachable, remaining follower nodes elect again a new leader among them

### ETCDCTL
- install ETCD on server
- download binary
```
wget -q --https-only "https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"
```
- unpack tar file
```
tar -xvf etcd-v3.3.9-linux-amd64.tar.gz
```
- add binary to `/usr/local/bin`
```
mv etcd-v3.3.9-linux-amd64/etcd/* /usr/local/bin/
```
- generate certificates
```
mkdir -p /etc/etcd/ /var/lib/etcd
# 
cp ca.pem kubernets-key.pem kubernetes.pem /etc/etcd/
```
- configure etcd service
```
ExecStart=/usr/local/bin/etcd \\
  ...
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster peer-1=https://${PEER1_IP}:2380,peer-2=https://${PEER2_IP}:2380 \\ # each etcd instance is peer 
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
  ...
```
- using `ÈTCDCTL` utility to store and retrieve data
- setting environment variable to API version of `etcdctl`
```
export ETCDCTL_API=3
```
- set a single key value pair with `key: name` and `value: john`
```
etcdctl put name john
```
- retrieve data
```
etcdctl get name
```
- get all keys
```
etcdctl get / --prefix --keys-only
```
