#### Commands

<details>
  <summary>command/cheat sheet</summary>
- create deployment  
- `k run first-deployment --image=nginx` (creates deployment **imperatively**)
-`k exec -it <pod-name> /bin/bash`
- `k get pods <pod_name>`:
  - ready 1/2 refers to containers in pod
  - look at `.spec.containers[].State` to see whats wrong
- `k edit deployments first-deployment` (use with caution, edits live objects and applies edit to object immediately)
- `k scale deployment first-deployment --replicas=3`
- `k get <resource> --show-labels`
- `k set image deployment/<your-deployment> nginx=nginx:1.9.1`
- `k create -f ....`
</details>


Types of command line tools:
`kubectl` 
`kubeadm` - to administer k8s to nodes?
`kubefed` - for federated clusters..
`kubelet...kube-proxy`

or...use client library
- clis send POST requests to kube api-server to instruct api-server what to do

- edit live deployments: `k edit deployment <deployment-name>`

---

#### Object Management methods
<details>
<summary>Imperative commands (suitable for deletion)</summary>
  - No yaml or config files
  - applies to objects that are already live
  - eg `k run/expose/autoscale`
  - eg `k create first-deployment --image=nginx`
</details>
<detail>
<sumamry>Imperative Object configuration</summary>
  - kubectl + yaml/config files
  - eg. `k create -f config.yml`, `k replace -f config.yml`...
</details>
<details>
<summary>Declarative Object config (preferred way...just use this..)</summary>
    - only YAML files used
    - `k apply -f config.yml`
    - Takes live obj config, current obj config, last applied obj config into consideration
</details>
- Don't mix and match!
- [kubectl create vs apply](https://stackoverflow.com/questions/47369351/kubectl-apply-vs-kubectl-create)

#### Namespaces
- Virtual cluster
- Provides scope for names...names need to be unique only inside a namespace
- Don't use namespaces for versioning..use labels

#### Secrets
- Create `secret` resource (from file / base64 string), mount secret as volume
##### From base64 string
- See [secrets.yml](./lab/secrets/secrets.yml) & [secrets-pod.yml](./lab/secrets/secrets-pod.yml)
<details>
<summary> Bash output </summary>

```sh
    k8s-101/k8s_on_the_cloud on  master [?] at ☸️  gke_parabolic-craft-216311_us-central1-a_my-first-clust
    er
    ➜ k create -f lab/secrets-pod.yml
    pod/secret-test-pod created

    k8s-101/k8s_on_the_cloud on  master [?] at ☸️  gke_parabolic-craft-216311_us-central1-a_my-first-clust
    er took 2s
    ➜ k exec secret-test-pod -- cat /etc/secret-volume/username
    user1234

    k8s-101/k8s_on_the_cloud on  master [?] at ☸️  gke_parabolic-craft-216311_us-central1-a_my-first-clust
    er took 3s
    ➜ k exec secret-test-pod -- cat /etc/secret-volume/password
    1234qwer

    k8s-101/k8s_on_the_cloud on  master [?] at ☸️  gke_parabolic-craft-216311_us-central1-a_my-first-clust
    er took 3s
    ➜
```
</details>


##### From file
- `kubectl create secret generic sensitive --from-file=./username.txt --from-file=./password.txt`
- see [secrets-pod-file.yml](./lab/secrets/secrets-pod-file.yml)
```bash
➜ k get secrets
NAME                  TYPE                                  DATA   AGE
default-token-79dd7   kubernetes.io/service-account-token   3      96m
sensitive             Opaque                                2      6s
test-secret           Opaque                                2      21m

➜ k describe secrets sensitive
Name:         sensitive
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password.txt:  14 bytes
username.txt:  14 bytes
```

<details>
<summary>Environment variables referenced by secretKeyRef are hidden in kubectl describe</summary>
  
```bash
➜ k describe pod secret-test-pod-file
Name:               secret-test-pod-file
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               gke-my-first-cluster-default-pool-1fe788fb-g00n/10.128.0.2
Start Time:         Sat, 07 Dec 2019 17:26:16 +0800
Labels:             <none>
Annotations:        kubernetes.io/limit-ranger: LimitRanger plugin set: cpu request for container test
-container
Status:             Running
IP:                 10.40.0.13
Containers:
  test-container:
    Container ID:   docker://591830164a760102fec811114665f259a79c46d95242f3b52c1dbd9164a011d5
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:189cce606b29fb2a33ebc2fcecfa8e33b0b99740da4737133cd
bcee92f3aba0a
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sat, 07 Dec 2019 17:26:18 +0800
    Ready:          True
mrk   Restart Count:  0
    Requests:
      cpu:  100m
    Environment:
      SECRET_USERNAME:  <set to the key 'username.txt' in secret 'sensitive'>  Optional: false
      SECRET_PASSWORD:  <set to the key 'password.txt' in secret 'sensitive'>  Optional: false
```
  </details>

#### Volumes
- see [here](../k8s_in_action/core_concepts/volumes.md)

- Create hardware specific Persistent Volume (PV) [manifest](./lab/volume-sample.yml)
- Create PV via PersistentVolumeClaim / Dynmic storage provisioning [link](../k8s_in_action/core_concepts/volumes.md#persistentvolumeclaim-with-dynamic-provisioning)

#### Configmaps
- see [configmaps](./lab/configmap) and [configmap-pod.yml](./lab/configmap/configmap-pod.yml)

#### Downward API vs Env vars
- downward API: used to pass k8s specific data from pod to container
  - expose information of a pod to its downward container
  - eg. labels, annotations can be mounted into a container via `volumes.downwardAPI` and the usual `spec.containers[].volumeMounts`
  - see [dapi-volume.yml](./lab/dapi-volume.yml)
  <details>  
    <summary> fieldRef gets mounted at `mountPath/path`</summary>  

    ```sh
      ➜ k exec -it dapi-volume /bin/bash
      root@dapi-volume:/# cd /etc/podinfo
      root@dapi-volume:/etc/podinfo# ls
      annotations  labels
      root@dapi-volume:/etc/podinfo# cat annotations
      build="two"
      builder="john-doe"
    ```
  </details>  


#### Container Lifecycle Events
- Exposed as hooks
- PostStart (called immediately after container is created, no params)
- PreStop (immediately before container terminates)
- Register hook handlers
  - Exec
  - HTTP
- atleast-once
<details>
  <summary>Place inside spec.containers[] of Pod template</summary>

```yaml
  lifecycle: 
    postStart:
      exec:
        command: ["/bin/bash", "-c", "poststart.sh"]
    preStop:
      exec:
        command: ["/bin/bash", "c", "prestop.sh"]
```
</details>

#### NodeSelectors and Affinity
- NodeSelectors:
  - Tag nodes with labesl, add nodeSelector to pod template
  - Add in spec.containers[]
- Affinity:
  - Node Affinity: steer pod to node
  - Pod Affinity: steer pods towards or away from pods
```yaml
nodeSelector:
  <some-node-label>: <some-node-label-value> 
```

#### Taints and Tolerance
- For anti-affinity between Pods and Nodes
- To make sure certain nodes never host certain nodes:
  - marks nodes with a taint
  - only pods with a tolerance for that taint are allowed to run on that node
- For Dedicated nodes for certain users
- Nodes with special hardware, reserve for pods that actually make use of hardware
- Taints
```sh
k taint nodes <node-name> env=dev:NoSchedule # key=value:effect, 
                                             # here its NoSchedule (don't schedule pods on this node 
                                             # unless pod has toleration for this key value pair)
```
- Tolerations
  - add in spec.containers[] of pod template
    ```yaml
    tolerations:
    - key: "dev"
      operator: "Equal"
      value: "env"
      effect: "NoSchedule"    # this pod has a toleration for this taint
    ```
#### Init Containers
- Run before app containers
- Always run-to-completion
- Run serially (each only starts after previous one finishes)
- example [init.yml](./lab/init.yml)

#### Pod Lifecycle
- see [here](../k8s_in_action/core_concepts/pods.md#pod-lifecycle)

#### Liveness & Readiness Probe
- see [here](../k8s_in_action/core_concepts/deploying_managed_pods.md#liveness-probes)
- Liveness
  -`spec.containers[].livenessProbe`
  - see [exec-liveness](./lab/exec-liveness.yml)
- Readiness
  - `spec.containers[].readinessProbe`
  - probe to determing if pod IP should be added to Endpoints object to be part of service

#### ReplicaSets
![Higher level K8s Objects][fig_2]
- Manifest contains:
  - pod template
  - pod selector
  - number of replicas
  - label of replica set
- see [frontend-replicaset](./lab/frontend-replicaset.yml)
- Delete just ReplicaSet but not its pods: `k delete --cascade=false`
    - pods are now orphans
    - can create new replicaSet and use labelSelector to include orphaned pods as part of 
      ReplicaSet
- Isolating pods from ReplicaSet
- Scaling a ReplicaSet
    - edit replicaset manifest and do `k apply -f manifest.yml`
- Auto-scaling a ReplicaSet [HorizontalAutoScaling](./lab/autoscalar.yml)
    - `--horizontal-pod-autoscaler-downscale-delay`

#### Deployments
<details>
  <summary> Encapsulates ReplicaSet, versioning, magic</summary>
- ReplicaSets associated with Deployment (encapsulates ReplicaSet template in Deployment)
- Rollback / Versioning
  - every change to deployment template is tracked (only for PodTemplate changes)
  - creates a new revision of the deployment for each change
  - `k rollout history deplyoment/nginx-deployment` to see reivison history
  - `k rollout undo deployment/nginx-deployment [--to-revision=]` rollback deployment
  - `.spec.revisionHistoryLimit` controls how many revisions (ReplicaSets) are kept
  - `k rollout status deployment deployment-name` see status of rollout
- Update container in pod template in deployment
  - new ReplicaSet and new pods created (running new version of container)
  - old ReplicaSet continues to exists but pods in old ReplicaSet gradually reduce to 0
- Magic
  - versioning
  - instant rollback
  - rolling deployments
  - blue/green
  - canary
- while creating deployments, append `--record`..keeps track of commands made 
- Fields
  - strategy
    - `.spec.stragety.type==Recreate`
    - `.spec.stragety.type==RollingUpdate`
- Pause/Resume Deployment
  - either imperatively or declaratively
  - `k rollout paus deployment/nginx-deployment` causes rolling update deployment to be paused
</details>
- see [nginx-deployment](./lab/nginx-deployment.yml)

#### Stateful sets
<details>
  <summary>Description</summary>
- Pods are created from the same spec, not interchangeable, each has persistent identifier that 
  is maintained across scheduling
- for ordered graceful deployment/scaling/deletion/termination/rolling updates
- storage needs to be Persistent
</details>
- see [sts](./lab/stateful-set.yml)
<details>
  <summary>Output</summary>

```bash

k8s-101/k8s_on_the_cloud/lab on  master [»!+] at ☸️  gke_parabolic-craft-216311_us-central1-a_my-fir
st-cluster
➜ k apply -f stateful-set.yml
statefulset.apps/web created

k8s-101/k8s_on_the_cloud/lab on  master [»!+] at ☸️  gke_parabolic-craft-216311_us-central1-a_my-fir
st-cluster took 2s
➜ k get pods -l app=nginx
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          12s
web-1   1/1     Running   0          9s

k8s-101/k8s_on_the_cloud/lab on  master [»!+] at ☸️  gke_parabolic-craft-216311_us-central1-a_my-fir
st-cluster
➜ k scale statefulset web --replicas=4
statefulset.apps/web scaled

k8s-101/k8s_on_the_cloud/lab on  master [»!+] at ☸️  gke_parabolic-craft-216311_us-central1-a_my-fir
st-cluster took 2s
➜ k get pods -l app=nginx
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          73s
web-1   1/1     Running   0          70s
web-2   1/1     Running   0          5s
web-3   0/1     Pending   0          3s


k8s-101/k8s_on_the_cloud/lab on  master [»!+] at ☸️  gke_parabolic-craft-216311_us-central1-a_my-fir
st-cluster took 2s
➜ k describe statefulset web
Name:               web
Namespace:          default
CreationTimestamp:  Tue, 10 Dec 2019 09:22:38 +0800
Selector:           app=nginx
Labels:             <none>
Annotations:        kubectl.kubernetes.io/last-applied-configuration:
                      {"apiVersion":"apps/v1","kind":"StatefulSet","metadata":{"annotations":{},"nam
e":"web","namespace":"default"},"spec":{"replicas":2,"select...
Replicas:           4 desired | 4 total
Update Strategy:    RollingUpdate
  Partition:        824639281820
Pods Status:        3 Running / 1 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Volume Claims:    <none>
Events:
  Type    Reason            Age   From                    Message
  ----    ------            ----  ----                    -------
  Normal  SuccessfulCreate  95s   statefulset-controller  create Pod web-0 in StatefulSet web succes
sful
  Normal  SuccessfulCreate  92s   statefulset-controller  create Pod web-1 in StatefulSet web succes
sful
  Normal  SuccessfulCreate  27s   statefulset-controller  create Pod web-2 in StatefulSet web succes
sful
  Normal  SuccessfulCreate  25s   statefulset-controller  create Pod web-3 in StatefulSet web succes
sful
```
</details>



[fig_2]: ./images/higher_level_k8s_objects.png

