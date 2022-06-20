# Scheduling

## Note on Resource Requirements and Limits
- setting default values for CPU and memory for PODs to pick up those values for containers
- default values for limit (maximum) and request (minimum)
- create a LimitRange for memory in the corresponding namespace: https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container
```

- create a LimitRange for CPU in the corresponding namespace: https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec: 
  limits:
  - default:
      cpu: 1
    defaultRequest:
      cpu: 0.5
    type: Container
```
- Resource: https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource

#### Important Note
- creating a container only specifying memory limit, but not its request:
  - container's memory request matches the defined limit
  - the container was not assigned the default memory request value
- creating a container only specifying memory request, but not its limit:
  -  container's memory request is set to specified value in container manifest
  -  container's memory limit is limited by default memory limit


## Note on editing Pods and Deployments
- you cannot edit speicifications of an existing pod other than:
  - spec.containers[*].image
  - spec.initContainers[*].image
  - spec.activeDeadlineSeconds
  - spec.tolerations
- 2 options to still edit a running pod:
  - `kubectl edit pod <pod name>`: opens the vi editor to edit the required properties
    - but changes are saved in file at temporary location
    - `kubectl delete pod <pod name>`
    - ``kubectl create -f /tmp/<temporary-file.yaml>`
  - ``kubectl get pod <pod name> -o yaml > new-pod.yaml`
    - `vi new-pod.yaml`: make changes in exported file
    - `kubectl delete pod <pod name>`
    - `kubectl create -f new-pod.yaml`
- in Deployments you can edit any field/property of the POD template, since with every change the deployment will automatically recreate the pod with the made changes
  - `kubectl edit deployment <deployment name>`: just edit the POD property of the deployment
