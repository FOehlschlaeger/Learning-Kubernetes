# Notes of this course chapter

---
# ETCD Commands

- `etcdctl` is CLI tool to interact with ETCD key value database
- `etcdctl` can interact with ETCD server using 2 API versions: version 2 and version 3
- **version 2 is default**
- each version has different sets of commands
- commands of one version do not work with other version and vice versa

### `etcdctl` v2
```commandline
etcdctl backup
etcdctl cluster-health
etcdctl mk
etcdctl mkdir
etcdctl set
```

### `etcdctl` v3
```commandline
etcdctl snapshot save
etcdctl endpoint health
etcdctl get
etcdctl put
```

### setting API version
```commandline
export ETCDCTL_API=3
```

### specify paths to authenticate to ETCD API server
```commandline
--cacert /etc/kubernetes/pki/etcd/ca.crt
--cert /etc/kubernetes/pki/etcd/server.crt
--key /etc/kubernetes/pki/etcd/server.key
```

### example
```commandline
kubectl exec etcd-master -n kube-system -- sh -c "ETCDCTL_API=3 \
etcdctl get / --prefix --key-only --limit=10 \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--cert /etc/kubernetes/pki/etcd/server.crt \
--key /etc/kubernetes/pki/etcd/server.key"
```

---
# API Version for different kind of Kubernetes Resources

| K8s resource | API Version |
| ------------ | ----------- |
| Pod          | v1          |
| Service      | v1          |
| ReplicaSet   | apps/v1     |
| Deployment   | apps/v1     |
