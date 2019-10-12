### Pods

[Introducing Pods](#introducing-pods)  
[Understanding Pods](#understanding-pods)  
[Pod Yaml Definition](#pod-yaml-definition)  
[Labels](#labels)  
[Commands](#commands)

***
#### Introducing Pods
- Pods are the central, most import concept in K8s. Everything else either manages, exposes, or is used by pods. 
- Instead of deploying containers individually, a **pod** of container(s) is deployed and operated on
- More common for pods to contain only a single container 
    - multiple containers? Usually when app consists of 1 main proc and other complementary proc (like a sidecar)
- If a pod contains multiple containers, they always run on a single worker node

#### Understanding pods
- K8s configures Docker to have all containers of a pod share the same set of Linux namespaces
- This enables containers of a pod to
    - share the same hostname and network interfaces -> share same ip address & port space
    - communicate through IPC 
- (since they share the same Network, UTS and IPC namespace, more on that)
- Containers **filesystem** comes from the container image. So..
     - By default, filesystem of each container is fully isolated from other containers of the same pod **unless** they share file directories via a **Volume**
***
- All containers in the same pod have the same loopback network interface; __they can communicate with each other through localhost__
- Take care not to bind containers in same pod to the same *port numbers*
- all pods in a k8s cluster reside in a single flat, shared, network-address space: **Every pod can access every other pod *at the other pod's IP address*** (without any NAT gateway, doesn't matter if pods are scheduled on different nodes)
- regardless of actual inter-node network topo, an additional software-defined network layered on top of the actual network enables each pod to get its own IP address and be accessible from all other pods through this additional network
***
- When organising containers into pods, if you need to scale a container individually -> That container should be deployed in a **separate** pod

#### Pod yaml definition
- Simple pod definition in yaml
```yaml
    apiVersion: v1               # Descriptor conforms to version v1 of k8s API
    kind: Pod                    # Type of k8s resource
    metadata:
      name: kubia-manual         # name of pod
      labels:                    
        env: prod                # labels attached to pod
        app: kubia
    spec:
      nodeSelector:              # nodeSelector tells k8s to deploy pod only to nodes containing gpu=true label
        gpu: "true"
      containers:
      - image: luksa/kubia       # container image to create container from
        name: kubia              # name of container
        ports:
        - containerPort: 8080    # port the app is listening on (purely informational, no effect if ommited)
          protocol: TCP
```


#### Labels
- arbitray **key-value** pairs attached to a **resource**
- they are utilized when selecting resources using *label selectors* 
    - resources are filtered based on whether they include the label specified in the selector
- usually attached when resource is created, but can be modified without needing to recreate the resource
- Label selectors: 
    - select subset of pods tagged with certain labels
    - perform operation on those pods
***
##### Constraining Pod Scheduling
- Sometimes there is a need to specify what type of node a pod should be scheduled on, 
this is done using labels as well (**nodeSelector** and node labels)
- Worker nodes can be categorised using labels as well (eg, nodes with GPU vs regular CPU)
- Possible to schedule pod on a *specific* node, but if that node fails, those pods will never be schedulable
    - each node has a unique label: *kubernetes.io/hostname=<actual_hostname_of_node>*
- Better to describe node requirements and let K8s select a node that matches those requirements

#### Namespaces
- Used to split resources into separate non-overlapping groups
- K8s namespaces provide a scope for object names
    - Instead of having all resources in 1 single namespace, split resources into multiple namespaces
    - allows reusing of resource names across different namespaces
- Especially useful in a multi-tenant environment (i.e. multiple groups sharing same cluster)
- Resource names only need to be unique within a namespace
- Only the **Node resource** is global and is *not* tied to a single namespace
- `dafault` namespace is used when no namespace specified in kubectl commands
- Current context's namespace / current context can be changed through `kubectl config`
***
##### Creating Namespace
```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: custom-namespace
```
- To craete an object in a namespace:
`kubectl create -f <some-file> -n custom-namespace`



#### Stopping Pods / Removing Pods
- Deleting pod = instructing k8s to terminate all the containers that are part of the pod
- K8s sends a `SIGTERM` signal to the process - waits for process (30s) to shut down *gracefully*
- If proc does not shut down in time, process is killed through `SIGKILL`
- To ensure processes are shut down gracefully, they need to handle `SIGTERM` signal properly

#### Commands
- `kubectl create -f <file-name>` used for creating any resource from a yaml or json file
- `kubectl get pod <pod-name>` get pod
    - `-o yaml`: get whole YAML definition of a pod
    - `--show-labels`
    - `-L <label1>,<label2>` list labels interested in
    - `-l label_name=some_name,label_name_2!=some_name_2` get resources with label_name=some_name and label_name_2!=some_name_2
    - `-n <namespace>` list pods in specified namespace
- `kubectl logs <pod-name> [-c container_name]` Get logs of container, -c optional if multiple containers 
    - `--previous` see logs of previous container (in case of restart)
- `kubectl portforward <pod-name> machine-local-port:pod-port` FOR DEBUGGING ONLY. Talk to specific pod without going through service
- `alias kcd='kubectl config set-context $(kubectl config current-context) -- namespace'` , then use `kcd some-namespace`
- `kubectl delete pod <pod-name1> <pod-name2>` delete pod
    - `--all` deletes all pods in current namespace, keeping current ns
    - deleting namespace removes all pods in namespace
- `kubectl delete all --all` deletes all(almost) resources in current namespace
