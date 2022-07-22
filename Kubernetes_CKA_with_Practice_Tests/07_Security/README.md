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

### Certifcate Generation using OpenSSL
- Certificate Authority
  - `ca.key`: generate keys: `openssl genrsa -out ca.key 2048`
  - `ca.csr`: create certificate signing request: `openssl req -new -key ca.key subj "/CN=KUBERNETES-CA" -out ca.csr` with component `/CN=` for which the certificate will be created
  - `ca.crt`: sign certificate: `openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt`
- Admin user
  - `admin.key`: generate keys: `openssl genrsa -out admin.key 2048`
  - `admin.csr`: create certificate signing request: `openssl req -new -key admin.key subj "/CN=kube-admin/O=system:masters" -out admin.csr` with component `/O=` for user groups
  - `admin.crt`: sign certificate: `openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt` to sign certificate with CA key pair for valid certificate within cluster
- same procedure for scheduler, controller-manager, kube-proxy
- other users besides `system:` clients can be assigned other user groups without admin privileges
- specify `--key`, `--cacert`, `--cert` into `kube-config.yaml`
```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
  name: kubernetes
    certificate-authority: ca.crt
    server: https://kube-apiserver:6443
users:
- name: kubernetes-admin
  user:
    client-certificate: admin.crt
    client-key: admin.key
```
- etcd server:
  - for HA define `--peer-...` certificates
- kube-apiserver:
  - kube-apiserver with several names to be addressed such as: `kubernetes`, `kubernetes.default`, `kubernetes.default.svc`, `kubernetes.defaul.svc.cluster.local`, IP addresses directly
  - `openssl genrsa -out apiserver.key 2048`
  - `openssl req -new -key apiserver.key subj "/CN=kube-apiserver" -out apiserver.csr -config openssl.conf` with `openssl.conf` containing DNS `subjectAltName`
  - `openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key -out apiserver.crt`
- kubelet server:
  - kubelet certificate and key on each node in cluster named after nodes
- kubelet client (talking to kube-apiserver):
  - each node with certifcate and CSR with node name (such that kube-apiserver knows which node communicates)
  - `system:node:node01` of group `SYSTEM:NOES`

### Checking Certificates Health
- cluster setup using `kubeadm`
  - xlsx file sheet: [kubernetes-certs-checker.xlsx](kubernetes-certs-checker.xlsx)
  - `/ect/kubernetes/manifests/kube-apiserver.yaml` to check all certificates of all components:
    - `--client-ca-file`
    - `--etcd-cafile`
    - `--etcd-certfile`
    - `--etcd-keyfile`
    - `--kubelet-client-certificate`
    - `--kubelet-client-key`
    - `--tls-cert-file`
    - `--tls-private-key-file`
  - details about certificate: `openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout` such as `Issuer`, `Validity.Not After`, `Subject`, `x509v3 Subject Alternative Name`
  - `kubectl logs etcd-master` to check for setup components as pods via `kubeadm` setup
  - without access of pods, use docker or container runtime engine and access logs of pods: `docker logs <container-ID>`

## [Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)
- new user creates key pair and signing request file such as `user.csr`
- create a new certificate signing request object in Kubernetes via `kubectl create -f csr.yaml`
- encode the content of the csr file: `cat user.csr | base64 | tr -d "\n"`
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: user-csr
spec:
  request: <output-of-base64-encoded-csr-file-of-user>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
```
- check existing csr objects: `kubectl get csr`
- approving a csr: `kubectl certificate approve user-csr`
- denying a csr: `kubectl certificate deny user-csr`
  - checking values in `spec.groups` to decide about denying or approving
- checkout csr objects: `kubectl get csr <csr-name> -o yaml`

## KubeConfig File
- stores information about `--server`, `--client-key`, `--client-certificate`, `--certificate-authority` to pass to each `kubectl` command
  - by default: `--kubeconfig config`
  - location: `$HOME/.kube/config`
- content sections:
  - clusters: clusters for different purposes such as development, production
  - contexts: connect user to cluster, for example `admin@production`
  - users: several users with different access and privileges on different clusters such as admin, developer etc.
- no new objects need to be created, only already existing objects can be called
```yaml
apiVersion: v1
kind: Config
current-context: my-kube-admin@my-kube-playground # add this to define default context of config file

clusters: # array to add several clusters
- name: my-kube-playground
  cluster:
    #certificate-authority: ca.crt # certificate of certificate authority
    certificate-authority-data: <base64-encoded-output-of-ca.crt-file> # use cat ca.crt | base64
    server: https://my-kube-playground:6443
    
contexts: # array to add several contexts
- name: my-kube-admin@my-kube-playground
  context:
    cluster: my-kube-playground
    user: my-kube-admin
    namespace: finance

users: # array to add several users
- name: my-kube-admin
  user: 
    client-certificate: admin.crt
    client-key: admin.key
```
- `kubectl config view`
- to change current context: `kubectl config use-context prod-user@production`
- better to input base64 encoded content of certificate files using `cat <file> | base64`
  - for decoding: `echo -n <string-to-decode> | base64 --decode`

## Authorization

### Roles and RoleBindings
- example for `Role`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: blue
rules:
- apiGroups:
  - apps
  resources:
  - pods
  - deployments
  verbs:
  - list
  - create
  - delete
  - get
```
- example for `RoleBinding`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-user-binding
  namespace: blue
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: developer
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: dev-user
```

## Image Security
- calling `image: nginx` in pod/deployment definition, is actually calling: `docker.io/library/nginx`
- use private image registry with credentials
- create secret with credentials
- **important note: `docker-registry` is a build-in secret type**
```commandline
kubectl create secret docker-registry regcred \
  --docker-server= private-registry.io \
  --docker-username= registry-user \
  --docker-password= registry-password \
  --docker-email= registry-user@org.com
```
- use credentials in pod/deploment definition:
```yaml
...
spec:
  containers:
  - name nginx
    image: private-registry.io/apps/internal-apps/nginx
  imagePullSecrets:
  - name: regcred
...
```
