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
