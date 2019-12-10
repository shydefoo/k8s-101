### Replication and other controllers: deploying managed pods
- Deployments should stay up and running without manual intervention
- To do this, **pods are never created directly**
- Other types of resources, eg **ReplicationControllers** or **Deployments** are created 
   - **these resources create and manage the actual pods**
#### Keeping pods healthy
- When a pod is scheduled to a node, the kubelet on node runs its containers (keeps them running as long as the pod exists)
- If container's main process crashes, the **kubelet** on the *node hosting the pod* will restart the container
    - K8s control plane components running on the maaster(s) have no part in this process
- Sometimes main process is up, but app stops responding (falls into infinite loop or deadlock), this is where **liveness probes** come in

#### Liveness probes
- K8s can check if a container is still alive through liveness probes
    - specified for each container in the pod's specification
- K8s (kublet) will periodically execute the probe and restart the container if the probe fails
- *Note: Different from readiness probe*
- Mechanisms to probe container:
    - HTTP Get probe:
        - performs HTTP get request on containers IP address, port and path
        - 2xx or 3xx code considered successful
    - TCP Socket probe:
        - tries to open TCP connection to specified port of container
        - if connection can be established, probe is successful
    - Exec probe:
        - executes arbitrary command inside the container and checks the command's exit status code. 
        - status code=0 means probe is successful
- Always remember to set an inital delay to account for app's startup time via *initialDelaySeconds*
- Liveness probes should be lightweight and snould not take too long to complete (as they are executed very frequently)
####  HTTP-based liveness probe
  ```yaml
        apiVersion: v1
        kind: pod
        metadata:
          name: some-pod
        spec:
          containers:
          - image: some-registry/some-image
            name: some-container
            livenessProbe:                     # liveness probe that will perform an HTTP GET
              httpGet:
                path: /                        # path to request in HTTP request
                port: 8080                     # network port the probe should connect to
              initialDelaySeconds: 15          # Kublet will wait 15s before executing first probe
  ``` 
- pod descriptor tells K8s to periodically perform HTTP GET requests on path / on port 8080 to determine if container is healthy
- When liveness probe fails, Kublet will restart container
- `kubectl describe pod <podname>` provides information of pod:
    ```yaml
        ...
        Containers:
          kubia:
            Container ID:       docker://480986f8
            Image:              luksa/kubia-unhealthy
            Image ID:           docker://sha256:2b208508
            Port:
            State:              Running                                         # container is currently running
              Started:          Sun, 14 May 2017 11:41:40 +0200                 
            Last State:         Terminated                                      # previous container terminated with exit code
              Reason:           Error                                           
              Exit Code:        137                                             
              Started:          Mon, 01 Jan 0001 00:00:00 +0000                 
              Finished:         Sun, 14 May 2017 11:41:38 +0200                 
            Ready:              True
            Restart Count:      1                                               # container has been restarted once
            Liveness:           http-get http://:8080/ delay=0s timeout=1s      
                                period=10s #success=1 #failure=3
            ...
        Events:
        ... Killing container with id docker://95246981:pod "kubia-liveness ..."
            container "kubia" is unhealthy, it will be killed and re-created.
      ```    
    - Exit code = sum of 2 numbers, 128 + x where x is the signal number sent to the process that caused it to terminate (eg 137=128+9), `SIGKILL` signal.
    - `delay=0s` shows probing begins immediately after container is started
    - `timeout=1s`, container must return a response in 1s or probe is counted as failure
    - `period=10s`, container probed every 10s
    - `failure=3`, container is restarted after probe fails 3 **consecutive** times
---

- Kubelet of the node hosting the pod ensures that containers are kept running. But if the node itself crashes, its the Control Plane that must create replacements for all the pods that went down with the node
    - Kubelet can't do anything if the node fails
- To ensure that the app is restarted on another node, the pod needs to be managed by another resource..eg. **Replication Resource**


#### Replication Controllers


- Resource that constantly monitors the list of running pods and makes sure the actual number of pods that match a certain label selector always matches the desired number
- Components:
    - label selector: determines what pods are in the ReplicationController's scope
    - replica count: specifies desired number of pods that should be running
    - pod template: used when creating new pod replicas
- Can be modified at any time
- Enables horizontal scaling, both manual and automatic
- Take note: a pod instance is never relocated to another node, a completely new pod is created to replace another instance
- Since RCs manage pods that match its label selector, a pod can be removed from or added to the scope of a RC by changing it's labels
    - useful when performing actions on a specific pod
#### Definition
  ```yaml
    apiVersion: v1
    kind: ReplicationController
    metadata:
      name: kubia                   # name of RC
    spec:
      replicas: 3                   # desired number of pod instances
      selector:
        app: kubia                  # pod selector determining what pods RC is operating on
      template:                     # pod template for creating new pods
        metadata:                   
          labels:
            app: kubia              # pod label here must match label selector of RC
        spec:
          containers:
          - name: kubia
            image: some-image/kubia
            ports:
            - containerPort: 8080
  ```
- Example description of RC
    ```yaml
      Name:           kubia
      Namespace:      default
      Selector:       app=kubia
      Labels:         app=kubia
      Annotations:    <none>
      Replicas:       3 current / 3 desired                                    1
      Pods Status:    4 Running / 0 Waiting / 0 Succeeded / 0 Failed           2
      Pod Template:
        Labels:       app=kubia
        Containers:   ...
        Volumes:      <none>
      Events:                                                                  3
      From                    Type      Reason           Message
      ----                    -------  ------            -------
      replication-controller  Normal   SuccessfulCreate  Created pod: kubia-53thy
      replication-controller  Normal   SuccessfulCreate  Created pod: kubia-k0xz6
      replication-controller  Normal   SuccessfulCreate  Created pod: kubia-q3vkg
      replication-controller  Normal   SuccessfulCreate  Created pod: kubia-oini2
    ```
    - `Pod Status: 4 Running` even though `Replicas: 3 current / 3 desires` because a pod that's terminating is still considered running, but not counted in the replica count


### ReplicaSets

- ReplicatSet behaves exactly like a ReplicationController, but with more expressive pod selectors
- ReplicationController's label selector only allows matching pods that include a certain label

#### Defining ReplicaSet


### DaemonSets
- Like ReplicaSet, but runs 1 single pod replica each node (or subset of nodes specified in nodeSelector)
- Usually used to run infra-related pods that perform system-level operations (eg. log collector, resource monitor)
  - K8s own kube-proxy process is needed to run on all nodes to make services work
- Example DaemonSet manifeest:
  ```yaml
    apiVersion: apps/v1beta2           # DaemonSets are inthe apps API group, version v1beta2
    kind: DaemonSet                    
    metadata:
      name: ssd-monitor
    spec:
      selector:
        matchLabels:
          app: ssd-monitor
      template:
        metadata:
          labels:
            app: ssd-monitor
        spec:
          nodeSelector:                # pod template includes a node selector, which 
            disk: ssd                  # selects nodes with the disk=ssd label
          containers:
          - name: main
            image: luksa/ssd-monitor
  ```
- ![DaemonSets run only a single pod replica on each node, ReplicaSets scatter tham around the whole cluster randomly][fig_4_8]

### Job Resource
- Creates pods that perform a single completable task
- ReplicationControllers, ReplicaSets, DaemonSets run continuous tasks that are never considered completed (Processes in such pods are restarted when they exit by the kubelet)
- In a completable task, after its process terminates, it should not be restarted again
- Node failure: pods on that node that are managed by a Job will be reschedule to other nodes
- Process failure: Job can be configured to restart the container or not
- Can be configured to:
  - run more than 1 pod instance `spec.completions`
  - run them in parrallel/sequentially `spec.parallelism`
  - limit a pod's time (if the pod runs longer than set time, system will try to terminate it, mark job as failed) `activeDeadlineSeconds` property in **pod spec**
  - retry x times before it is marked as failed by specifying `spec.backoffLimit` (default 6)
- ![Pods managed by Jobs are rescheduled until they finish successfully][fig_4_10]

#### Job resource definition
- ```yaml
  apiVersion: batch/v1            # Jobs are in the batch API group, version v1
  kind: Job
  metadata:
    name: batch-job
  spec:                           # No pod selector here, pods will be created based on labels in pod template
    completions: 5                # Ensure 5 pods completed successfully
    parallelism: 2                # Up to 2 pods can run in parallel
    backOffLimit: 5
    template:
      metadata:
        labels:
          app: batch-job
      spec:
        restartPolicy: OnFailure  # Jobs can't use the default restart policy (Always), need to state explicity
        containers:
        - name: main
          image: registry/image

  ```

### CronJob Resource
- like a cron job in unix, create a **Job** resource according to the Job template which creates and starts pod replicas according to the Job's pod template
- Job resources will be created from the CronJob resource at approx the scheduled time
- Job then creates pods
- Ideally, CronJob creates only a single Job for each execution...but sometimes:
  - 2 Job are created at the same time..so
    - Ensure jobs are **idempontent**
  - No Job created at all..so
    - Ensure next job run performs any work that should have been done by previous (missed) run

#### CronJob definition
- ```yaml
  apiVersion: batch/v1beta1
  Kind: CronJob
  metadata:
    name: batch-cron-job
  spec:
    schedule: "0,15,30,45, * * * *"   # job runs at 0, 15, 30, 45 min of every hour everyday
    jobTemplate:
      spec:
        template:
          metadata:
            labels:
              app: periodic-batch-job
          spec:
            metadata:
              labels:
                app: periodic-batch-job
            spec:
              restartPolicy: OnFailure
              containers:
              - name: main
                image: registry/image
  ```




[fig_4_8]: ./images/04fig08_alt.jpg
[fig_4_10]: ./images/04fig10_alt.jpg
