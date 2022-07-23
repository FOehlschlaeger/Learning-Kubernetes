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

