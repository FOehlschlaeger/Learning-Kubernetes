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
