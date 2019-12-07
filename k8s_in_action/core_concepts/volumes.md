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
- `gcePersistentDisk, awsElasticBlockStore, azureDisk`: for mounting cloud provider-specific storage
- `persistentVolumeClaim`: a method to use pre/ dynamically provisioned persistent storage

#### So..
- Best way to attach persistant storage to a pod is to only create the PersistantVolumeClaim and the pod
- EVERYTHING else is taken care of by the *dynamic PersistantVolume provisioner
![Complete picture of dynamic provisioning][fig_6_10]









[fig_6_10]: ./images/06fig10_alt.jpg "Figure 6.10 The complete picture of dynamic provisioning of PersistantVolumes"
