# Cluster Maintenance

## References

### Kubernetes Software Versions
- https://kubernetes.io/docs/concepts/overview/kubernetes-api/
- https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md
- https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md

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
