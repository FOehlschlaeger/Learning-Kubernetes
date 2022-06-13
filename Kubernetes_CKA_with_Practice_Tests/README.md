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

#### Examples
- create nginx pod: `kubectl run nginx --image=nginx`
- generate pod manifest yaml file without actual creation: `kubectl run nginx --image=nginx --dry-run=client -o yaml`
- create deployment: `kubectl create deployment --image=nginx nginx`
- generate deployment yaml file without actual creation: `kubectl create deployment --image=nginx nginx --dry-run=client -o yaml`
- generate deployment yaml file with 4 replicas without actual creation: `kubectl create deployment --image=nginx nginx --replicas=4 --dry-run=client -o yaml > nginx-deployment.yaml`
- create deployment from yaml file: `kubectl create -f nginx-deployment.yaml`

