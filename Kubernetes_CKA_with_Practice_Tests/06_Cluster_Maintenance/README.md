# Cluster Maintenance

## References
### Kubernetes Software Versions
- https://kubernetes.io/docs/concepts/overview/kubernetes-api/
- https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md
- https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md

### Backup and Restore
- https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster
- https://github.com/etcd-io/website/blob/main/content/en/docs/v3.5/op-guide/recovery.md
- https://www.youtube.com/watch?v=qRPNuT080Hk

---
## Upgrading Cluster

Example upgrading process to version 1.20.0: https://v1-20.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

### Upgrade master nodes / controlplane
#### Upgrade `kubeadm`
```
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.20.0-00 && \
apt-mark hold kubeadm
```

#### Check upgraded version
```
kubeadm version
```

#### Check upgrading before actual starting upgrade process
```
kubeadm upgrade plan
```

#### Upgrade `kubeadm` on controlplane
```
sudo kubeadm upgrade apply v1.20.0
```

#### Make controlplane node unschedulable
```
kubectl drain controlplane --ignore-daemonsets
```

#### Upgrade `kubelet` and `kubectl`
```
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.20.0-00 kubectl=1.20.0-00 && \
apt-mark hold kubelet kubectl
```

#### Restart kubelet daemon on controlplane node
```
sudo systemctl daemon-reload
```
```
sudo systemctl restart kubelet
```

#### Make controlplane schedulable again
```
kubectl uncordon controlplane
```

### Upgrade Worker Node
#### Drain worker node (`kubectl` commands are executed on controlplane node)
```
kubectl drain node01 --ignore-daemonsets
```

#### Connect to worker node
```
ssh root@<WORKER-IP>
```

#### Upgrade `kubeadm` on worker node
```
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.20.0-00 && \
apt-mark hold kubeadm
```

#### Upgrade configuuration on worker node
```
sudo kubeadm upgrade node01
```

#### Drain worker node (on controlplane node)
```
kubectl drain node01 --ignore-daemonsets
```

#### Upgrade `kubelet` and `kubectl`
```
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.20.0-00 kubectl=1.20.0-00 && \
apt-mark hold kubelet kubectl
```

#### Restart kubelet daemon on worker node
```
sudo systemctl daemon-reload
```
```
sudo systemctl restart kubelet
```

#### Make worker node schedulable again
```
kubectl uncordon node01
```

#### Check status of cluster
```
kubectl get nodes
```

---
## Cluster Backup and Restore

### Best Practice
- Use a version control system such as Git/GitHub to store configuration files for Kubernetes deployments and resources

### Backup all configurations running in cluster
```
kubectl get all --all-namespaces -o yaml > all-deploy-services.yaml
```

### Backup Tools
- ARK / [Velero](https://velero.io/docs/v1.9/)

### ECTD Cluster
- in `etcd` the state of cluster is stored (the status information shown in `kubectl get` command)
- usually etcd database hosted on master nodes
- data storage location defined during etcd configuration, for example by `--data-dir=/var/lib/etcd`

**Important Note**
- `etcdctl` is command line client for etcd
- make sure to set correct API version: `ETCDCTL_API=3`
- Using `??TCDCTL` always define for each command 
  - `--endpoints=https://127.0.0.1:2379`: default as ETCD is running on master node and exposed on localhost port 2379
  - `--cert=.../etcd-server.crt`: identify secure client using this TLS certificate file
  - `--key=.../etcd-server.key`: identify secure client using this TLS key file
  - `--cacert=.../ca.crt`: verify certificates of TLS-enabled secure servers using CA bundle

#### Create Backup
- taking a snapshot of etcd database in current directory
```
ETCDCTL_API=3 etcdctl snapshot save snapshot.db
```
##### Example:
```
ETCDCTL_API=3 etcdctl snapshot save /opt/snapshot-pre-boot.db --cert=/etc/kubernetes/pki/etcd/server.crt --cacert=/etc/kubernetes/pki/etcd/ca.crt --key=/etc/kubernetes/pki/etcd/server.key --endpoints=127.0.0.1:2379
```

- view status of backup
```
ETCDCTL_API=3 etcdctl snapshot status snapshot.db
```

#### Restoring from ETCD Backup
- stop kube-apiserver service
```
service kube-apiserver stop
```
- restore from snapshot (from backup a new cluster is created, `--cacert`, `--cert`, `--key`, `--endpoints` are not necessary due to no interaction with apiserver when restoring from backup file)
```
ETCDTL_API=3 etcdctl snapshot restore snapshot.db --data-dir=/var/lib/etcd-from-backup
```
##### Example
```
ETCDCTL_API=3 etcdctl snapshot restore --data-dir=/var/lib/etcd.db /opt/snapshot-pre-boot.db
```
- initilization of a new cluster configuration from backup file
- new members of etcd are configured as new members of new cluster to prevent joining of a new member to an existing cluster
- `--data-dir` defines the new data location to store etcd data of new cluster inside `etcd` pod
- use new data directory at `volumes.hostPath.path` entry in etcd configuration file, usually at `/etc/kubernetes/manifests/etcd.yaml` (`hostPath` refers to local path, `mountPath` and `--data-dir` are locations in `etcd` pod, so `volumeMounts.mountPath` for etcd-data should match `--data-dir` entry)
- reload the service daemon
```
systemctl daemon-reload
```
- restart etcd service
```
service etcd restart
```
- start kube-apiserver service
```
service kube-apiserver start
```

