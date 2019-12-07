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

### Volumes & Secrets
- Create `secret` resource (from file / base64 string), mount secret as volume
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

