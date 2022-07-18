# Security

## References

## Security Primitives
- Kube-API-Server is first point of security due to central unit 
- Authenticaion: Who can access the cluster?
  - Files: Username and Passwords
  - Files: Username and Tokens
  - Certificates
  - External Authentication Provides: LDAP
  - Service Accounts for machines
- Authorization: What can they do?
  - RBAC Authorization
  - ABAC Authorization
  - Node Authorization
  - Webhook Mode
- all communication is secured by TLS encryption
- by default: all pods can access each other pod
- Network policies to define which pod is allowed to access other pods

## Authentication
- several cluster users: Admins, Developers, Application End Users, Bots/Machines (Third Party)
- securing management access to cluster
- securing access of internal processes
- Application End Users are covered by security measures by applications itself
- K8s does not manage user accounts natively
- `kubectl get serviceaccount`
- Kube-APIserver authenticates users accessing before command processing by `kubectl` or API directly `curl http://kube-server-ip:6443/`
- Auth mechanisms
  - static password file (not recommended, and deprecated from v1.19)
    - csv file with `password,username,user-ID` via `--basic-auth-file=user-details.csv` in kube-apiserver.service with flag in `ExecStart=/usr/local/bin/kube-apiserver` of in kube-apiserver Pod at `/etc/kubernetes/manifests/kube-apiserver.yaml`
  - static token file (not recommended, and deprecated from v1.19)
    - csv file with `token,username,user-ID,group` via `--token-auth-file=user.details.csv` to kube-apiserver
  - certificates
  - identity services (LDAP, Cerberos)
- RBAC for new users

## SSL TLS certificates
- TLS certificates ensure encrypted communication and identities
- asymmetric encryption concept
  - private and public keys
  - encrypt data with public key
  - encrypted data can only be decrypted with private key
  - private key must not be shared!
  - ssh access after generating key pair with public and private key: `ssh-keygen`
  - add public key of user to `~/.ssh/authorized_keys` file on server
- certificates are used to ensure valid source and trusted origin of public key
  - issue and signature verify legitimation of certificates
  - self-signed certificate (not-save)
  - certificate authority: organization for signing and validation of certificates after sending a certificate signing request (CSR)
- public key files ending with `.crt` or `.pem`
- private key files ending with `.key` or `-key.pem` (substring `key` in filename)

## TLS in Kubernetes
- inter-node communication needs to be encrpyted
- user-node communication must enable TLS encryption
- server certificates for servers and client certificates for clients and users
- server certificates
  - kube-api server
  - etcd server
  - kubelet server
- client certificates
  - admin user with admin.crt and admin.key for `kubectl `REST API requests to kube-apiserver
  - kube scheduler
  - kube-controller-manager
  - kube-proxy
- at least one certificate authority (CA) must exist for cluster with `ca.crt` and `ca.key`
- 

## Authorization

