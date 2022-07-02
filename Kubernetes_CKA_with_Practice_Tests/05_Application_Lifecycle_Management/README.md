# Application Lifecycle Management

## References

## Rolling Updates and Rollbacks
- Create a deployment
```
kubectl create -f deployment-definition.yaml
```

- List existing deployments
```
kubectl get deployments
```

- Update deployments
```
kubectl apply -f deployment-definition.yaml
```
or to update without remaining change in yaml manifest
```
kubectl set image deployment/myapp-deployment nginx=nginx:1.9.1
```

- Rollout status
```
kubectl rollout status deployment/myapp-deployment
```

- Show applied rollouts of deployment
```
kubectl rollout history deployment/myapp-deployment
```

- Rollout undo to rollback to previous deployment version
```
kubectl rollout undo deployment/myapp-deployment
```


## [Secrets](https://kubernetes.io/docs/concepts/configuration/secret)
- Kubernetes secrets encode data in `base64` format, so it is not very safe, since encoding is not encryption
- Best practices:
  - not checking-in secret object definition files to source code repositories
  - enabling encryption at Rest for secrets so they are stored encrypted in etcd: https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
- safety mechanisms of Kubernetes:
  - a secret is only sent to a node if a pod on that node requires it
  - kubelet stores the secret into a tmpfs so that the secret is not written to disk storage
  - once the pod that depends on the secret is deleted, kubelet will delete its local copy of the secret data as well
- safer ways to store secret data
  - Helm Secrets: HashiCorp Vault


## Multi-Container Pods Desing Patterns
- sidecar pattern
- adapter pattern (discussed in CKAD exam)
- ambassador pattern (discussed in CKAD exam)


## [InitContainers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
- in a multi-container pod, each containers lifecycle is expected to run a process that stays alive as long as the pod's lifecycle
- if any of the containers inside of a pod fails, the pod restarts
- sometimes a process should run to completion, for example at initialization of the pod: solution **initContainers**
- **initContainer** configuration
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp.pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'git clone <some-repository-used-by-application> ; done;']
```
- process of initContainer needs to finish before actual other containers are started to be executed
- in case of multiple initContainers, the container array defined in yaml manifest are run one at a time in sequential order
- if any of the initContainers fail to complete, the pod is started repeatedly until the initContainer succeed
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp.pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2 ; done;']
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```
