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
- ```yaml
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
    - ```yaml
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
***

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
- ```yaml
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
    - ```yaml
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
