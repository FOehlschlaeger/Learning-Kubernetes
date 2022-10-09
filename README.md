# Learning-Kubernetes
This project aims to learn Kubernetes to prepare for the [CKA certification](https://www.cncf.io/certification/cka/) and [CKAD certification](https://www.cncf.io/certification/ckad/) based on a local cluster consisting of virtual machines. 

## Prerequisites:
- Linux CLI skills
- `vi`/`nano` as editor for yaml files
- `grep`/`awk`/`head`/`tail`/`cat` for providing output from commands 
- `scp` for copying files from one server to another
- `tmux` for running commands on more than one server at a time
- `systemd`/`journalctl` for debugging clusters and changing systemd unit files etc.
- experience with `docker` to run standard containers, used commands to keep docker containers running, understanding images, tags etc.

## Resources
### Links and Tutorials
- https://kubernetes.io/docs/
- https://kubernetes.io/docs/tasks/
- https://github.com/kelseyhightower/kubernetes-the-hard-way
- https://github.com/cncf/curriculum

### Udemy

- https://github.com/kodekloudhub/certified-kubernetes-administrator-course

#### CKA
The Udemy course [Certified Kubernetes Administrator with practice tests](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/) is also used in terms of understanding Kubernetes and preparation for the certification.

#### CKAD
Topic in CKAD
- multi-container pods design patterns
  - ambassador pattern
  - adapter pattern
- Self-healing applications
  - self-healing applications through ReplicaSets and Replication Controllers
  - Replication Controller: ensures that pod is recreates automatically and ensures that enough replicas are running at all times
  - Liveness and Readiness Probes
- Storage
  - StatefulSets
- Service Accounts

---
## Kubernetes Command Line Tools

#### `kubectx`
- 

#### `kubens`
- 

---
## Cluster Architecture Setup

- which VMs?
- what network in VirtualBox?
- what servers to use?
- ...

## Learning and Understanding Kubernetes

- install container runtimes on cluster servers
- setting up local cluster
- setup network tool for kubernetes
- ...

