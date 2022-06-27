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
