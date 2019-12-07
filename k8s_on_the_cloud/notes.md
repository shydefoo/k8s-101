#### Commands

- create deployment  
`k run first-deployment --image=nginx` (imperative command, creates deployment)

`k exec -it <pod-name> /bin/bash`

- `k get pods <pod_name>`:
  - ready 1/2 refers to containers in pod
  - look at `.spec.containers[].State` to see whats wrong

- `k edit deployments first-deployment` (use with caution, edits live objects and applies edit to object immediately)
- `k scale deployment first-deployment --replicas=3`


Types of command line tools:
`kubectl` 
`kubeadm` - to administer k8s to nodes?
`kubefed` - for federated clusters..
`kubelet...kube-proxy`

or...use client library
- clis send POST requests to kube api-server to instruct api-server what to do

- edit live deployments: `k edit deployment <deployment-name>`

#### Object Management methods
- Imperative commands (suitable for deletion)
  - No yaml or config files
  - applies to objects that are already live
  - eg `k run/expose/autoscale`
  - eg `k create first-deployment --image=nginx`
- Imperative Object configuration
  - kubectl + yaml/config files
  - eg. `k create -f config.yml`, `k replace -f config.yml`...
- Declarative Object config (preferred way...just use this..)
    - only YAML files used
    - `k apply -f config.yml`
    - Takes live obj config, current obj config, last applied obj config into consideration
- Don't mix and match!
- [kubectl create vs apply](https://stackoverflow.com/questions/47369351/kubectl-apply-vs-kubectl-create)

#### Objects
- identified using:
  - names: client-given (name field in metadata of manifest)
  - uid: system-generated

#### Namespaces
- Virtual cluster
- Provides scope for names...names need to be unique only inside a namespace
- Don't use namespaces for versioning..use labels

### Secrets
- Create `secret` resource (from file / base64 string), mount secret as volume
#### From base64 string
- See [secrets.yml](./lab/secrets.yml) & [secrets-pod.yml](./lab/secrets-pod.yml)
- ```bash
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

#### From file
- `kubectl create secret generic sensitive --from-file=./username.txt --from-file=./password.txt`
- see [secrets-pod-file.yml](./lab/secrets-pod-file.yml)
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
```
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

#### Configmaps
- see [configmaps](./lab/configmap) and [configmap-pod.yml](./lab/configmap/configmap-pod.yml)
