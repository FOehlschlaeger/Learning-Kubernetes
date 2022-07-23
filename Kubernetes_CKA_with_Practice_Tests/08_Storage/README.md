# Storage

## References
- https://kubernetes.io/docs/concepts/storage/persistent-volumes/#claims-as-volumes

---
## Volumes and Mounts
- Docker containers are of transient and ephemeral nature, meaning if a container is finished or disrupted by error, the internal data is deleted along with the container itself
- for data persistence, a volume is attached to container to store the processed data of the container in that volume, independent of the lifetime of an attached container 
- in Kubernetes, mount volumes to pods, such that the container inside the pod can access the mounted volume to store the data persistently
- mount the volume to a directory inside the container
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
spec: 
  containers:
  - image: alpine
    name: alpine
    command: ["/bin/sh", "-c"]
    args: ["shuf -i 0-100 -n 1 >> /opt/number.out;"]
    volumeMounts: # array of mounted volumes inside the container
    - mountPath: /opt
      name: data-volume
  volumes: # array of volumes available for containers
  - name: data-volume
    hostPath: # volume type: path of host of the pod (node)
      path: /data
      type: Directory
```
- option `hostPath` is not recommended in a multi-node cluster setup since synchronization of all volume directories on all nodes is expected
- external options:
  - AWS via
  ```
  ...
  awsElasticBlockStore:
    volumeID: <volume-id>
    fsType: ext4
  ```

## Persistent Volumes (PVs)
- cluster-wide pool of storage which can be used by applications without changing the application/pods configuration each time
- using storage from this pool via ´persistentVolumeClaim´
- Kubernetes resource: `PersistentVolume` using ´kubectl create -f <pv-file.yaml>´
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes: # defines how the volume should be mounted on host, as `ReadOnlyMany`, `ReadWriteOnce`, `ReadWriteMany`
    - ReadWriteOnce
 capacity: 
   storage: 1Gi
 hostPath: # not for production
   path: /tmp/data
```

## Persistent Volume Claims (PVCs)
- `PersistentVolume` and `PersistentVolumeClaims` are two separate objects in Kubernetes namespace
- admin creates a set of `PersistentVolumes`
- user creates `PersistentVolumeClaims` for applications
- each PVC is bound to a single PV based on properties of PVs
- bounding based on sufficient storage capacity, access modes, volumes modes, storage class based on claim
- using `labels` and `selectors` in case of several PVs matching the PVC
- PVC will remaining in `pending` state if no matching PVs are available
- `pvc-definition.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: 
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```
- after creating the PVC via `kubectl create -f pvc-definition.yaml` all available PVs are searched for matching resources, access modes etc.
- to delete: `kubectl delete persistentvolumeclaim myclaim`
  - by default: `persistentVolumeReclaimPolicy: Retain`, meaning the PV will remain until manually deletion by administrator, not available for reuse by other PVCs
  - `persistentVolumeReclaimPolicy: Delete`: PV deleted together with deleted PVC
  - `persistentVolumeReclaimPolicy: Recycle`: data in volume will be scrubbed to be reused again (**deprecated, use [dynamic provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/) instead**)
- if PVC is deleted which is still in use by a pod, the PVC will not be deleted but in state `Terminating`, just when deleting the pod using the PVC, will terminate the PVC at the end
- using a PVC in a pod definition (or ReplicaSets or Deployments by adding this to the pod template section):
```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: mypod
    spec:
      containers:
        - name: myfrontend
          image: nginx
          volumeMounts:
          - mountPath: "/var/www/html"
            name: mypd
      volumes:
        - name: mypd
          persistentVolumeClaim:
            claimName: myclaim
```

## Example
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-log
spec:
  capacity:
    storage: 100Mi
  accessModes: 
  - ReadWriteMany
  hostPath:
    path: /pv/log
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim-log-1
spec:
  accessModes: 
  - ReadWriteMany
  resources: 
    requests:
      storage: 50Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: webapp
  namespace: default
spec:
  containers:
  - env:
    - name: LOG_HANDLERS
      value: file
    image: kodekloud/event-simulator
    imagePullPolicy: Always
    name: event-simulator
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-7vb7k
      readOnly: true
    - mountPath: /log
      name: pv-1
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: controlplane
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: pv-1
    persistentVolumeClaim:
      claimName: claim-log-1
  - name: kube-api-access-7vb7k
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
---
```

## [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- **not part of CKA or not relevant to CKA exam regarding to Udemy Course, but relevant for CKAD**
- StatefulSet is the workload API object used to manage stateful applications.
- Manages the deployment and scaling of a set of Pods, and provides guarantees about the ordering and uniqueness of these Pods.
- Like a Deployment, a StatefulSet manages Pods that are based on an identical container spec. Unlike a Deployment, a StatefulSet maintains a sticky identity for each of their Pods. These pods are created from the same spec, but are not interchangeable: each has a persistent identifier that it maintains across any rescheduling.
- If you want to use storage volumes to provide persistence for your workload, you can use a StatefulSet as part of the solution. Although individual Pods in a StatefulSet are susceptible to failure, the persistent Pod identifiers make it easier to match existing volumes to the new Pods that replace any that have failed.

## Storage Classes
- dynamic provisioning of storage needed by applications
- storage needed by applications is provided automatically at an exernal provider such as AWS, GCE, etc.
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: google-storage
provisioner: kubernetes.io/gce-pd
parameters: # additional parameters to pass to provisioner such as type of storage etc.
  type: pd-standard
  replication-type: None
```
- using a `StorageClass` a `PersistentVolume` is not needed, since the volume is created automatically based on the `StorageClass`
- binding a `StorageClass` object to a PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec: 
  accessModes:
  - ReadWriteOnce
  storageClassName: google-storage
  resources:
    requests:
      storage: 500Mi
```
- calling the PVC in pod definition section
```yaml
...
spec:
  containers:
  - image: alpine
    name: alpine
    ...
    volumeMounts:
    - mountPath: /opt
      name: data-volume
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: myclaim
```
- the `StorageClass` still creates a PV but automatically, not by administrator before creating the PVC
- **parameters of provisioners are provisioner-dependent**
