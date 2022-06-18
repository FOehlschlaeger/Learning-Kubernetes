# Exam Preparation

## Important Links for [CKA](https://www.cncf.io/certification/cka/)

from [Udemy course](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/)

### Exam
- Exam Curriculum Topics: https://github.com/cncf/curriculum
- Candidate Handbook: https://www.cncf.io/certification/candidate-handbook
- Exam Tips: http://training.linuxfoundation.org/go//Important-Tips-CKA-CKAD
- CKA FAQ: https://www.cncf.io/certification/cka/faq/
- kubectl conventions: https://kubernetes.io/docs/reference/kubectl/conventions/

### Udemy course KodeKloud Labs
- Link to KodeKloud Labs of Udemy course: https://kodekloud.com/courses/labs-certified-kubernetes-administrator-with-practice-tests/

### Udemy course documents
- Udemy Course Curriculum: https://github.com/mmumshad/kubernetes-the-hard-way
- Additional documentation, answers to practice questions: https://github.com/kodekloudhub/certified-kubernetes-administrator-course

---
## Certification Tip
- use `kubectl run` command to generate yaml manifests

### Examples
#### Pod
- create nginx pod: 
```
kubectl run nginx --image=nginx
```
- generate pod manifest yaml file without actual creation: 
```
kubectl run nginx --image=nginx --dry-run=client -o yaml
```

#### Deployment
- create deployment: 
```
kubectl create deployment --image=nginx nginx
```
- generate deployment yaml file without actual creation: 
```
kubectl create deployment --image=nginx nginx --dry-run=client -o yaml
```
- generate deployment yaml file with 4 replicas without actual creation: 
```
kubectl create deployment --image=nginx nginx --replicas=4 --dry-run=client -o yaml > nginx-deployment.yaml
```
- create deployment from yaml file: 
```
kubectl create -f nginx-deployment.yaml
```
- scaling a deployment: 
```
kubectl scale deployment nginx --replicas=4
```

#### Service
- create service named `redis-service` of default type `ClusterIP` to expose pod redis on port `6379`: 
```
kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml
```
- with usage of selector `app=redis`: 
```
kubectl create service clusterip redis --typ=6379:6379 --dry-run=client -o yaml
```
- **You cannot pass in selectors as an option, so generate file and modify it before create service**
- create a service named `nginx` of type `NodePort` to expose pod nginx's port `80` on port `30080` on the nodes: 
```
kubectl expose pod nginx --type=NodePort --port=80 --name=nginx-service --dry-run=client -o yaml
```
- or (not preferred, since selectors need to be set): 
```
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml
```

### Imperative commands with `kubectl`
- `--dry-run=client`: option will not create the resource but will tell if resource could be created
- `--dry-run=client -o yaml`: option will output the resource definition in yaml format without actual creating the resource


