# Troubleshooting

## References
- [Troubleshooting Applications](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Troubleshooting Cluster](https://kubernetes.io/docs/tasks/debug/debug-cluster/)
- [Troubleshooting kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/)
- [Kubelet Integration and Configuration](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/)
- [Debug Service Issues](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/)
- [DNS Troubleshooting](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)

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
### CNI plugins
- CNI plugins are used to setup network
- kubelet config parameters for executing plugins
  - `cni-bin-dir`: kubelet probes this directory for plugins on startup
  - `network-plugin`: network plugin to use from `cni-bin-dir`, is must match the name of a plugin from the directory
- kubelet uses the configuration files in lexicographic order, in case of multiple CNI configuration files in the directory
- to install WeaveNet without provided URL, see in documentation at https://kubernetes.io/fr/docs/setup/production-environment/tools/kubeadm/high-availability/#%C3%A9tapes-pour-le-premier-n%C5%93ud-du-control-plane

Plugins:
- WeaveNet suports kubernetes network policies
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
- Flannel, so far, does not support kubernetes network policies
```
kubectl apply -f               https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
```
- Calico, most avanced CNI network plugin
```
curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

### DNS
- `CoreDNS` as extensible DNS server as cluster DNS
- CoreDNS memory usage is predominantly affected by the number of pods and services in the cluster
- Kubernetes resources for `CoreDNS`
  - service account `coredns`
  - cluser-roles `coredns` and `kube-dns`
  - clusterolebindings `coredns` and `kube-dns`
  - deployment `coredns`
  - configmap `coredns`
  - service `kube-dns`
- CoreDNS deployment with `Corefile plugin` is defined as configmap
- Port `53` is ued for DNS resolution
- backend for k8s for `cluster.local` and reverse domains
```
...
  kubernetes cluster.local in-addr.arpa ip6.arpa {
    pods insecure
    fallthrough in-addr.arpa ip6.arpa
    ttl 30
  }
...
```
- forward out of cluster domains directly to right authoritative DNS server
```
proxy . /etc/resolv.conf
```

### Troubleshooting CoreDNS
If `CoreDNS` pods are in `pending` state
- check if network plugin is installed

If `CoreDNS` pods have `CrashLoopBackOff` or Error State
- if nodes are running on SELinux with an older Docker version
  - upgrade to a newer version of docker, or
  - disable SELinux, or
  - modify the `coredns` deployment to set `allowPrivilegeEscalation=true`, or
- cause for `CrashLoopBackoff` might be when a `CoreDNS` pod deployed in Kubernetes detects a loop

Work arounds:
- add this flag to kubelet config yaml: `resvolvconf. <path-to-your-real-resolv-conf-file>`
  - this flag tells kubelet to pass an alternate `resolv.conf` to pods
  - for systems using `systemd-resolved`, the location of real `resolv.conf` is typically `/run/systemd/resolve/resolv.conf`
- disable the local DNS cache on host nodes and restore `/etc/resolv.conf` to the original
- edit the `Corefile`, replacing forward `. /etc/resolv.conf` with the IP address of your upstream DNS, for example `. 8.8.8.8`. But this will only fix the issue for CoreDNS, kubelet will continue to forward the invalid `resolv.conf` to all default dnsPolicy Pods, leaving them unable to resolve DNS

If `CoreDNS` pods and `kube-dns` service is working fine, check the `kube-dns` service has valid endpoints: 
```
kubectl -n kube-system get ep kube-dns
```
- if there are no endpoints, inspect the service an make sure it uses the correct selectors and ports

### Kube-Proxy
- network proxy that runs on each node
- `kube-proxy` maintains network rules on nodes, allowing network communication to pods from network sessions inside or outside of the cluster
- cluster setup with `kubeadm` tool: `kube-proxy` is deployed as `DaemonSet`
- `kube-proxy` is responsible for watching services and endpoints associated with each service
- `kube-proxy` is responsible for sending the traffic to actual pods if clients connect to service using virtual IPs
- `kubectl describe ds kube-proxy -n kube-system` runs binary with following command in kube-proxy container:
```
Command:
  /usr/local/bin/kube-proxy
  --config=/var/lib/kube-proxy/config.conf
  --hostname-override=$(NODE_NAME)
```
- configuration is fetched from configuration file `/var/lib/kube-proxy/config.conf`
- hostname can be overwitten by hostname of at which the pod is running
- in config file the folloing values are defined:
  - `clusterCIDR`
  - kubeproxy mode
  - `ipvs`
  - `iptables`
  - `bindaddress`
  - `kube-config`

Troubleshooting
- check if `kube-proxy` pod in namespace `kube-system` is running
- check `kube-proxy` logs
- check if configmap is correctly defined and if config file for running `kube-proxy` binary is correct
- check `kube-config` is definedin configmap
- check `kube-proxy` is running inside the container
```
netstat -plan | grep kube-proxy
```


### Possible cases
If no valid IP addresses are assigned or are there no valid endpoints of the services
- in `kubernetes.io/docs/`: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network
- install WeaveNet plugin via `kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml`

If the `kube-proxy` in `kubectl -n kube-system get pod` is not working
- check the logs of the pod via `kubectl -n kube-system logs <kube-proxy pod>`
- check description of the `kube-proxy` daemonset
- check values in configmap of `kube-proxy` using `kubectl -n kube-system get cm kube-proxy -o yaml` or `kubectl -n kube-system describe cm kube-proxy`
- compare parameters between daemonset and the configmap of `kube-proxy` for invalid, unmatching values, such as wrong filenames of config files (**in the configmap the filename is one of the first parameter, not the listed values**)
- update and edit the daemonset via `kubectl -n kube-system edit ds kube-proxy`
- delete running pod to recreate the pod using the updated parameters via `kubectl -n kube-system delete pod <kube-proxy-pod-name>`

