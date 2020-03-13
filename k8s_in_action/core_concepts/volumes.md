### Volumes: Attaching disk storage to containers
- Volumes are used to provide either temporary or persistant storage to a pod's containers
- Temporary storage:
  - Volumes are defined as part of a pod, sharing the same lifecycle (vol is created/destroyed when pod is started/deleted)
  - New container will see files written to volume by previous container *WITHIN THE SAME POD*
- Persistant Storage:
  - Persist data even when pod(s) is scheduled to a different node

#### Volume Types:
- `emptyDir`: simple empty directory used for storing transient data
- `hostPath`: Used for mounting directories from worker node's filesystem into pod (more relevant for DaemonSets)
- `gitRepo`: Volume initialized by checking out contents of a Git repository
- `nfs`: NFS share mounted into the pod
- `configMap`, `secret`, `downwardAPI`: special types of volumes used to expose K8s resources / cluster info to the pod
- `gcePersistentDisk, awsElasticBlockStore, azureDisk`: for mounting cloud provider-specific storage
- `persistentVolumeClaim`: a method to use pre/ dynamically provisioned persistent storage

#### Basic Structure
- `spec.containers[].`
```yaml
  volumeMounts:
    - name: html
      mountPath: /path/dir
```
- `spec.`
```yaml

  volumes:
    - name: html # specify vol name
      <type_of_vol>: ...
```

#### Using Persistent Storage
* For apps that run inside a pod and need to persist data to disk, but have that same data avaialable when pod is scheduled 
  to another node, **Network Access Storage** (NAS) required (mount external storage inside volume)
* Eg. GCE Persistent Disk as a pod volume, pods will see same data no matter which node it is scheduled on
* Persistent volume used depends on underlying infra & possible cluster provider
* For self hosted cluster, **NFS** share would work
* Pods being aware of their underlying infrastructure is an antipattern

#### PersistentVolumes and PersistentVolumeClaims
![Pv & PVCs][fig_6_6]
* cluster admin sets up underlying storage (eg NFS) 
* registers underlying storage with k8s by creating a PersistentVolume resource (through K8s API)
  * specify size & access mode supported
* User creates PersistentVolumeClaim when pod requires persistent storage
  * specify minimum size & access mode
* K8s finds appropriate PersistentVolume and **binds volume to claim**
* PersistentVolumeClaim is used as a volume inside pod

* Create PersistentVolume (usually done by administrator)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
  capacity:                                 1
    storage: 1Gi                            1
  accessModes:                              2  
  - ReadWriteOnce                           2  can EITHER be mounted by single client for reading and writing
  - ReadOnlyMany                            2  OR mounted by multiple clients for reading only 
  persistentVolumeReclaimPolicy: Retain     3  after claim is released, PersistentVolume should not be erased/deleted
  gcePersistentDisk:                        4  backed by GCE Persistent Disk
    pdName: mongodb                         4
    fsType: ext4                            4
```

* Create PersistentVolumeClaim (k8s user)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc              1
spec:
  resources:
    requests:                    2
      storage: 1Gi               2
  accessModes:                   3
  - ReadWriteOnce                3 support single client (performing both read and writes)
  storageClassName: ""           4
```

* <details><summary>Using PersistantVolumeClaim inside pod</summary>
  
  - Best way to attach persistant storage to a pod is to only create the `PersistentVolumeClaim` as a `spec.volumes` and **mount** volume in `spec.containers[].volumeMounts`
  - PersistentVolumeClaim
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: my-persistent-vol-claim
    spec:
      resources:
        requests:
          storage: 1Gi
      accessModes:
        - ReadWriteOnce
    ```
  - Use PersistentVolumeClaim in Pod (cluster will dynamically provision PersistentVolume and underlying storage)
      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: mongodb
      spec:
        containers:
          - name: mongo
            image: mongodb
            volumeMounts:
              - name: mongodb-vol
                mountPath: /data/db
        volumes:
          - name: mongodb-vol
            persistentVolumeClaim:
              claimName: my-persistent-vol-claim

      ```
    </details>

* RWO, ROX, RWX - ReadWriteOnce, ReadOnlyMany, ReadWriteMany (pertain to number of **worker nodes** that can use the volume 
at the same time, NOT number of pods)

#### PersistentVolumeClaim with dynamic provisioning
* Create a **PersistentVolume provisioner** and define 1 or more **StorageClass** objects to let users choose what type of PersistentVolumes they want
* let system create new PersistentVolume each time a PV is requested through a PVC

- EVERYTHING else is taken care of by the *dynamic PersistantVolume provisioner  

![Complete picture of dynamic provisioning][fig_6_10]

* specifiying empty string as storage class name ensures the PVC binds to a pre-provisioned PV instead of dynamically
provisioning a new one









[fig_6_10]: ./images/06fig10_alt.jpg "Figure 6.10 The complete picture of dynamic provisioning of PersistantVolumes"
[fig_6_6]: ./images/06fig06_alt.jpg "Figure 6.6 Persistent Volumes are provisioned by cluster admins and consumed by pods through PersistentVolumeClaims"
