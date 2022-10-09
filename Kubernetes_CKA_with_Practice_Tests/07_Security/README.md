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
- pki = public key infrastructure

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

## API Groups
- all kubernetes resources are grouped ito different API groups
- top level API group: core API group
- then: named API group with one for each section, within there are the resources, each of which having specific verbs, such as `/apps` with `/v1`, with `/deployments`, `/replicasets` etc.
- `/deployments` with verbs: `list`, `get`, `create`, `delete`, `watch`, `update`, etc.
- accessing kubernetes API via `curl` does not give all information since no authentication information was specified
```
curl http://localhost:6443 -k
```
- accessing kubernetes API via `curl` and authentication information
```
curl http://localhost:6443 -k
  --key admin.key
  --cert admin.crt
  --cacert ca.crt
```
- alternatively, access kubernetes API via `kubectl proxy` uses Kubernetes certificates from `$HOME/.kube/config` to proxy to Kubernetes API with all information
```
kubectl proxy # open port on 127.0.0.1:8001
```
- access the started proxy
```
curl http://localhost:8001 -k
```

## Authorization

- `RBAC` = Role Based Access Control
- Create a new `Role` 
- Connect user to new role via `Rolebinding`
- `Role` and `Rolebinding` fall under scope of `namespaces` 

### Roles and RoleBindings
- example for `Role`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: blue
rules:
- apiGroups: # api (default core api group), apis, rbac.authorization.k8s.io, network.k8s.io, [""] for all apiGroups
  - apps
  resources: # [""] for all resources
  - pods
  - deployments
  verbs: # [""] for all verbs
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

### Checking access
```
kubectl auth can-i create deployments
```
- check as another user and in specific namespace
```
kubectl auth can-i create deployments --as dev-user --namespace prod
```

### Restrict access to specific resources
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: blue
rules:
- apiGroups: # api (default core api group), apis, rbac.authorization.k8s.io, network.k8s.io, [""] for all apiGroups
  - apps
  resources: # [""] for all resources
  - pods
  - deployments
  verbs: # [""] for all verbs
  - list
  - create
  - delete
  - get
  resourceNames: ["blue", "orange"] # restrict acess to pods "blue" and "orange"
```

## Service Accounts
- service accounts are accounts for non-human services or applications interacting with kubernetes API
- creating a service account creates a secret, which stores the token of the service account for authentication to be authorized to query the kubernetes API
```
kubectl create serviceaccount <name-of-sa>
```
- `Token` refers to a secret created at service account creation, storing the actual token information
```
kubectl describe serviceaccount <name-of-sa
```
- this token has to be used by external application to authenticate to kubernetes API as `bearer token`
```
kubectl describe secret <secret-in-token-of-sa>
```
- for applications run in kubernetes cluster themselves, the secret with service account token can be mounted as secret volume in application pod
- default service account is mounted automatically to each new created pod, at `Mounts:` path available inside the pod
- token of default service account is in JWT format, without expiration date (which causes security issues)
- mounting a new service account to pod, the pod definition needs to be updated at `spec:serviceAccountName=<new-sa-name>`
- you can not change the serviceaccount of an existing pod, so the pod needs to be recreated, in deployment a new rollout will be initiated after changing the service account in deployment definition
- avoid automatically mounting of default service account via: `autoMountServiceAccountToken: False` in pod definition
- since `v1.22` the `TokenRequestAPI` is used to create a token with expiring date due to security issues, mounted as `projected volume` into the pod 
- since `v1.24` the token is no longer created automacically when creating a service account, and needs to be created manually
```
# create a new serviceaccount
kubectl create serviceaccount <name-of-sa>
# create a new token belonging to that new service account
kubectl create token <name-of-sa>
```
- this created token has a expiring date of one hour after creation (default)
- to create a service account with token without expiring date, create service account first and afterwards the secret (**not recommended, only without accessto TokenRequestAPI**)
```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
annotations:
  kubernetes.io/service-account.name: <name-of-sa>
```

## Image Security
- calling `image: nginx` in pod/deployment definition, is actually calling: `docker.io/library/nginx`
- use private image registry with credentials
- create secret with credentials
- [**important note: `docker-registry` is a build-in secret type**](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
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

## [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- Docker runs containers as `root` users by default, but `root` user in container has restricted capabilities compared to `root` user of host
```
docker run --user=1000 ubuntu sleep 3600 # run container with process user id 1000, ps aux
```
- definition in Dockerfile
```Dockerfile
FROM ubuntu

USER 1000
...
```
- settings can be set on container and on pod
- settings on container will override settings of pod
- a security context defines privilege and access control settings for a Pod or Container
- add `securityContext` in pod or container definition
- **`Capabilities` are only available in `securityContext` section in container definition**
- container level:
```yaml
...
spec:
  containers:
  - name: sec-ctx-4
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
      runAsUser: 1000 #root
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]
...
```
- pod level:
```yaml
...
spec:
  securityContext:
    runAsUser: 1000 #root
  containers:
  - name: sec-ctx-4
    image: gcr.io/google-samples/node-hello:1.0
...
```

## Network Policies
- define Network Policies to restrict/enable/disable the default "all allow" Kubernetes communication between all deployed pods
- Network Policies are objects in Kubernetes namespace
- rules to avoid communication between specific microservices
- only allows traffic for configured ports
- link Network Policy to one or more pods via `Labels` and `Selectors`
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from: 
    # selector rules and ipBlock are OR correlated
    # selector rules within array are AND correlated
    # both pod and namespace label must match
    - podSelector: # allow pods with this label
        matchLabels:
          name: api-pod
      namespaceSelector: # namespace must be labeled beforehand
        matchLabels:
          name: prod
    - ipBlock: # allow specific IP
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 3306
```
- in this example (difference in `- namespaceSelector`) there are three rules which are OR related, meaning only one of them needs to match:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from: 
    - podSelector: # allow pods with this label
        matchLabels:
          name: api-pod
    - namespaceSelector: # namespace must be labeled beforehand
        matchLabels:
          name: prod
    - ipBlock: # allow specific IP
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 3306
```
- example with `ingress` and `egress`, to push a backup to a bakup server with IP `192.168.5.10`
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress # adding Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
    ports:
    - protocol: TCP
      port: 3306
  egress: # new egress section
  - to: 
    # any rules can be used such as `podSelector`, `namespaceSelector` or `ipBlock`
    - ipBlock: # allow specific IP
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 80
```
- another example calling `selector` labels of services such as `name=mysql` of service `db-service` via `kubectl get svc db-service`
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-policy
spec:
  podSelector:
    matchLabels:
      name: internal
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector: 
            matchLabels:
              name: mysql
      ports:
        - protocol: TCP
          port: 3306
    - from: 
        - podSelector:
            matchLabels:
              name: payroll
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels: 
              name: mysql
      ports:
        - protocol: TCP
          port: 3306
    - to:
        - podSelector:
            matchLabels:
              name: payroll 
      ports:
        - protocol: TCP
          port: 8080
```
