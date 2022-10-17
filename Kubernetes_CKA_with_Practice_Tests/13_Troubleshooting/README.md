# Troubleshooting

## References
- [Troubleshooting Applications](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Troubleshooting Cluster](https://kubernetes.io/docs/tasks/debug/debug-cluster/)
- [Troubleshooting kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/)

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
- check all control plane components / pods if working correctly
```
kubectl get pod -A
#
kubectl -n kube-system get pod
```
- check all parameters and commands inside of control plane components if configured correctly, such as `--kubeconfig`

## Worker Node Failures
- 

## Networking Failures
- 
