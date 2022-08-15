# Design and Install a Kubernetes Cluster

## References
- [Youtube Channel: Setup Kubernetes the Hard Way](https://www.youtube.com/watch?v=uUupRagM7m0&list=PL2We04F3Y_41jYdadX55fdJplDvgNGENo)
- [Github Repo: Setup Kubernetes the Hard Way](https://github.com/mmumshad/kubernetes-the-hard-way)

---
## Cluster Design
- Local virtual cluster on one machine
- Virtualbox
- CNI: Weave Network
- 1 Loadbalancer
- 3 Master Nodes with stacked topology (ETCD on master nodes)
- 2 Worker Nodes

## Kubernetes Infrastructure

## Configuration of High Availability

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
