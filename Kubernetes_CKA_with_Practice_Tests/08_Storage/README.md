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
