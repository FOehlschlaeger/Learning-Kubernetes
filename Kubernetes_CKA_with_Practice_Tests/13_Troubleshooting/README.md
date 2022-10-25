# Troubleshooting

## References
- [Troubleshooting Applications](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Troubleshooting Cluster](https://kubernetes.io/docs/tasks/debug/debug-cluster/)
- [Troubleshooting kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/)
- [Kubelet Integration and Configuration](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/)

## Application Failure
Possible reasons and causes for application failures to check while troubleshooting:
- false names of services, pods, deployments
- wrong target ports of services to direct traffic to pods
- wrong or missing labels of pods to be referenced by services
- wrong or missing environment variables necessary for applications to run
- false defined selectors in service definitions
- all possible combinations of the previous mentioned causes

### Show many information for all ressources in all namespaces
```
kubectl get all --show-labels -o wide -A
```

### Check defined environment variables in running pods
```
kubectl -n <namespace> exec -it <pod> -- env
```

### Fast replacement of pods after failed editing
```
# kubectl -n <namespace> edit pod <pod-name> fails at changes in specific parameters, gets stored in /tmp/ directory
kubeclt -n <namespace> replace --force -f /tmp/<filename-from-edit-command>
```

## Control Plane Failures
- if cluster is setup using `kubeadm` tool
- check all control plane components / pods if working correctly
```
kubectl get pod -A
#
kubectl -n kube-system get pod
```
- check all parameters and commands inside of control plane components if configured correctly, such as `--kubeconfig`
- check if all mounted volumes (i.e. `hostPath`) in pod or deployment are correct and existing, valid directories
- **on master nodes**: check running controlplane services
```
# kube-apiserver
service kube-apiserver status
# kube-controller-manager
service kube-controller-manager status
# kube-scheduler
service kube-scheduler status
```
```
systemctl status kube-apiserver
systemctl status kube-controller-manager
systemctl status kube-scheduler
```
- **on worker nodes**: check running node services
```
# kubelet
service kubelet status
# kube-proxy
service kube-proxystatus
```
```
systemctl status kubelet
systemctl status kube-proxy
```
- check logs of running controlplane components
```
kubectl logs kube-apiserver-master -n kube-system
# or use hosts logging solution
sudo journalctl -u kube-apiserver
```

## Worker Node Failures
- Check node details
- Check `Conditions = [True | False | Unknown]` of node that indicate status of nodes via
```
kubectl describe node <node-name>
```
- Condition status set to `Unknown` when worker node has no connection to master node
- to bring node back up if crashed, check memory and CPU usage via ´top´ and `df -h`
- check status of `kubelet` and `kubelet logs`
```
service kubelet status
#
systemctl status kubelet
#
sudo journalctl -u kubelet
```
- check kubelet certificates for validity
```
openssl x509 -in /var/lib/kubelet/worker-1.crt -text
```

### Possible cases
If worker node is not part of the Kubernetes cluster, i.e. not appearing via `kubectl get nodes`: 
- stopped `kubelet` service
  - check the status of services on the nodes, for example: `systemctl status kubelet` 
  - check the service logs using `journalctl -u kubelet`
  - if the service is stopped, start the stopped service directly on the worker or run `ssh <missing-node> "service kubelet start"` or via `systemctl start kubelet`
- error in `kubelet` service
  - check error messages via `journalctl -u kubelet`
  - maybe, miss configuration in config file of `kubelet`
  -path to kubelet config file with contexts, clusters etc.: `/etc/kubernetes/kubelet.conf`
  - fix parameter in kubelet service config file: `/var/lib/kubelet/config.yaml`
  - restart service on node: `systemctl restart kubelet`
  - check if node is correctly working and integrated into Kubernetes cluster
- kubelet service is running, but error in configuration
  - check via `journalctl -u kubelet`
  - error such as `connection refused`, such as wrong address and port of API server
  - fix error in `kubeconfig` at `/etc/kubernetes/kubelet.conf` o node
  - restart `kubelet` service on node
  - check if node is correctly working and integrated into Kubernetes cluster

## Networking Failures
If no valid IP addresses are assigned or are there no valid endpoints of the services
- in `kubernetes.io/docs/`: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network
- install WeaveNet plugin via `kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml`

If the `kube-proxy` in `kubectl -n kube-system get pod` is not working
- check the logs of the pod via `kubectl -n kube-system logs <kube-proxy pod>`
- check description of the `kube-proxy` daemonset
- check values in configmap of `kube-proxy` using `kubectl -n kube-system get cm kube-proxy -o yaml` or ``kubectl -n kube-system describe cm kube-proxy`
- compare parameters between daemonset and the configmap of `kube-proxy` for invalid, unmatching values, such as wrong filenames of config files (**in the configmap the filename is one of the first parameter, not the listed values**)
- update and edit the daemonset via `kubectl -n kube-system edit ds kube-proxy`
- delete running pod to recreate the pod using the updated parameters via `kubectl -n kube-system delete pod <kube-proxy-pod-name>`

