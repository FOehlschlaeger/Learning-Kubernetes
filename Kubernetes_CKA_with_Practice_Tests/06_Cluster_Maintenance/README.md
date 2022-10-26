# Cluster Maintenance

## References
### Kubernetes Software Versions
- https://kubernetes.io/docs/concepts/overview/kubernetes-api/
- https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md
- https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md

### Backup and Restore
- https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster
- https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#restoring-an-etcd-cluster
- https://github.com/etcd-io/website/blob/main/content/en/docs/v3.5/op-guide/recovery.md
- https://www.youtube.com/watch?v=qRPNuT080Hk
- https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/

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
- if `etcdctl` is not installed on client, ssh into etcd pod or install a client on local machine via
```
apt-get instal etcd-client
```
- make sure to set correct API version: `ETCDCTL_API=3`
- Using `ÈTCDCTL` always define for each command 
  - `--endpoints=https://127.0.0.1:2379`: default as ETCD is running on master node and exposed on localhost port 2379
  - `--cert=.../etcd-server.crt`: identify secure client using this TLS certificate file
  - `--key=.../etcd-server.key`: identify secure client using this TLS key file
  - `--cacert=.../ca.crt`: verify certificates of TLS-enabled secure servers using CA bundle

### How is the data stored in ETCD database?
- See also: https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
- Data is encrypted when written to ETCD
- Check if newly created secret is encrypted
```
ETCDCTL_API=3 etcdctl \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt   \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key  \
   get /registry/secrets/default/secret1 | hexdump -C
```
- Shown output should be prefixed with `k8s:enc:aescbc:v1:`
- Verify the correct decryption when retrieved via API by `kubectl get secret secret1 -n default -o yaml`
- Secret should show the base64-encoded string as value
- **Important: In the hexdump output the plain secret key is visible!**

#### Enabling Encryption of ETCD data
- Check if encryption is enabled
```ps aux | grep kube-apiserver```
- Check for the flag `--encryption-provider-config`
- If not configured:
  - Create a new `EncryptionConfiguration`, for example:
```
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources: # define which resources should be encrypted
      - secrets
    providers: # ordered list of different provider options to enable decryption, first one is used for decryption
      - aescbc:
          keys: # keys are used for the actual encryption
            - name: key1
              secret: <BASE 64 ENCODED SECRET> # value generated by command: 'head -c 32 /dev/urandom | base64'
      - identity: {} # default provider for encryption, i.e. no encryption is provided
```
  - Store that configuration file at `/etc/kubernetes/enc/enc.yaml` on the controlplane node
  - Pass this configuration as argument to kube-apiserver manifest as mounted hostPath volume
```
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 10.10.30.4:6443
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    ...
    - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml  # <-- add this line
    volumeMounts:
    ...
    - name: enc                           # <-- add this line
      mountPath: /etc/kubernetes/enc      # <-- add this line
      readonly: true                      # <-- add this line
    ...
  volumes:
  ...
  - name: enc                             # <-- add this line
    hostPath:                             # <-- add this line
      path: /etc/kubernetes/enc           # <-- add this line
      type: DirectoryOrCreate             # <-- add this line
  ...
```
  - Restart kube-apiserver

### Update the encryption of all secrets
- Secrets are encrypted at storing in ETCD
- Previously stored secrets after enabling encryption, are not automatically encrypted afterwards
- To apply encryption to all secrets on server side new via updating the secrets, run
```
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

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

#### Check if `etcd` is external
- switch context in case of setup with several kubernetes clusters, visible in `$HOME/.kube/config`
```
kubectl config use-context <context-name>
```
- check running pods in namespace `kube-system` and files at `/etc/kubernetes/manifests` on controlplane for static pods
- check running processes `ps aux | grep etcd` for process `kube-apiserver` and the config referring to external etcd store

#### Locate default data directory of external etcd store
- identify IP address of external etcd server via `ps aux | grep etcd` on controlplane node of cluster with external etcd store
- switch to external etcd server
- check running processes with `ps aux | grep etcd`

#### Check members of `etcd` cluster
- ssh to external etcd server
```
ETCDCTL_API=3 etcdctl \
 --endpoints=https://127.0.0.1:2379 \
 --cacert=/etc/etcd/pki/ca.pem \
 --cert=/etc/etcd/pki/etcd.pem \
 --key=/etc/etcd/pki/etcd-key.pem \
  member list
```

#### Backup etcd store of kubernetes cluster with stacked etcd store
- get `pki` credentials information to create backup
```
kubectl describe  pods -n kube-system etcd-cluster1-controlplane  | grep advertise-client-urls
```
```
kubectl describe  pods -n kube-system etcd-cluster1-controlplane  | grep pki
```
- log into controlplane node with `etcdctl` installed and create a snapshot/backup via `snapshot save`
```
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key snapshot save ./cluster1.db
```
- copy snapshot from controlplane node to local node into target directory
```
scp <controlplane-ip>:/root/cluster1.db /opt/
```

### Restore backup of etcd store of cluster with external etcd store
- copy existing backup file to external etcd server
```
scp /opt/cluster2.db etcd-server:/root/
```
- use `etcdctl` to restore snapshot with new `data-dir` location
```
ETCDCTL_API=3 etcdctl snapshot restore --endpoints=https://<etcd-ip>:2379 --cacert=<etcd-cacert> --cert=<etcd-cert> --key=<etcd-keyfile> --data-dir=<new-location> <snapshot-file.db>
```
for example:
```
ETCDCTL_API=3 etcdctl snapshot restore --endpoints=localhost:2379 --key=/etc/kubernetes/pki/etcd/etcd-key.pem --cacert=/etc/kubernetes/pki/etcd/etcd.pem --cert=/etc/kubernetes/pki/etcd/ca.pem --data-dir=/var/lib/etcd-data-new cluster2.db
```
- update systemd service unit by adding the new location for `data-dir`
```
nano /etc/systemd/system/etcd.service
```
- check permissions of user on new `data-dir`
```
chown -R etcd:etcd <new-data-dir-location>
```
- Reload and restart the `etcd` service
```
systemctl daemon-reload
```
```
system restart etcd
```
- recommended to restart controlplane components (kube-scheduler, kube-controller-manager, kubelet)

