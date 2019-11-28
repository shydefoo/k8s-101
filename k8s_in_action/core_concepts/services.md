### Services: Enabling Clients to Discover and talk to pods

- [Intro](#introduction)
- [Service Definition](#service-definition)
- [Discovering Services](#discovering-services)
- Service Types:
  - [ClusterIP (default)](#service-definition)
  - [ExternalName](#connecting-to-services-living-outside-cluster)
  - [NodePort](#nodeport)
  - [LoadBalancer](#load-balancer)
- [Endpoint Definition](#endpoint-definition)
- [Ingress Definition](#ingress-definition)

#### Introduction
- K8s Service is a resource to create a single, constant point of entry to a group of pods providing the same service
- Each service has an IP address and port that **never changes** while the service exists
  - Pods are ephemeral
  - IP address is assigned to a pod after pod has been scheduled to a node (clients can't know the ip address of the pod up front)
  - Clients shouldn't care how many pods are backing the service and what their IPs are
  - With the service, all those pods are accessible through a single IP address
  - Eg: ![Frontend/Backend Service][fig_5_1]


#### Service Definition
```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: some-app
  spec:
    sessionAffinity: ClientIP # redirect all requests originating 
                              # from same client IP to same pod (either None or ClientIP)
    ports:
    - name: http
      port: 80            # port this service will be available on
      targetPort: 8080    # container port the service will be forwarded to
    - name: https         # for multiport service
      port: 443
      targetPort: https   # using named ports, if port in pod definition has name https
                          # allows changing of port numbers without having to change service spec
    selector:
      app: some-app       # all pods with the app=some-app label will be part of this service
``` 

- with that, an internal cluster ip is assigned to the service `some-app`:
```
  $ kubectl get svc
  NAME         CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
  kubernetes   10.111.240.1     <none>        443/TCP   30d
  some-app     10.111.249.153   <none>        80/TCP    6m   // service
```  
- service is a clusterIp service (by default), so only accessible from inside the cluster
![routing of request by service][fig_5_3]


#### Discovering Services
- via:
  - Envinronment Variables
  - DNS

##### Environment Vairables
- When a pod is started, K8s initializes a set of env vars pointing to each service that **exists at that moment**
- If service is created before client pods, processes in those pods can get IP address and port of the service by inspecting their env vars
```
  $ kubectl exec some-app-3inly env
  PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
  HOSTNAME=some-app-3inly
  KUBERNETES_SERVICE_HOST=10.111.240.1
  KUBERNETES_SERVICE_PORT=443
  ...
  SOME_APP_SERVICE_HOST=10.111.249.153      # Cluster ip of service
  SOME_APP_SERVICE_PORT=80                  # Port service is available on
  ...
```
- Note: Dashes in service names are converted to underscores, all letters are uppercased when service name is used as the prefix in the env var's name


##### Discovering service through DNS
- k8s includes a `kube-dns` pod in the `kube-system` namespace
- the pod runs a DNS server, which all other pods running in the cluster are automatically configured to use
  - via modifiying each container's `/etc/resolv.conf` file
- any DNS query performed by a process running in a pod will be handled by K8s' own DNS server, which **knows all the services running in your system**
- Each service gets a DNS entry in the internal DNS server, and client pods that know the name of the service can access it through the *fully qualified domain name* (FQDN)
  - example FQDN: `some-app.default.svc.cluster.local`
    - `some-app`: service name
    - `default`: namespace (can be ommited if pods are in same namespace)
    - `svc.cluster.local` is a configurable cluster domain suffix used in all cluster local service names, 
- Client must still know service's port number (either through a standard port eg port 80 or through env vars)
- Note: curl-ing service works, but **pinging service does not**
  - service's cluster IP is a virtual IP, hence only has meaning when combined with a service port

#### Connecting to services living outside cluster
- Instead of having service redirect connections to pods in the cluster, require services to redirect connections to **external IP(s) and port(s)
- Do this by manually creating a `Service` and `Endpoints` resource
![connecting to services outside cluster][fig_5_4]

##### Service Endpoints
- Services don't link to pods directly, instead the `Endpoints` resource sits between the service and pod
```
  $ kubectl describe svc some-app
  Name:                some-app
  Namespace:           default
  Labels:              <none>
  Selector:            app=some-app                                           1
  Type:                ClusterIP
  IP:                  10.111.249.153
  Port:                <unset> 80/TCP
  Endpoints:           10.108.1.4:8080,10.108.2.5:8080,10.108.2.6:8080        2 
  Session Affinity:    None
  No events. 
```
- see 2, list of pod IPs and ports that represent the endpoints of this service
- Endpoints resource is a list of IP addresses and ports exposing a service
- Pod selector in service spec is used to build a list of IPs and ports, which is stored in the Endpoints resource
- Service proxy selects one of those IP and port pairs and redirects incoming connections to the pod listening at that location
- To create a service with manually managed endpoints, need to create both Service and Endpoints resource
- Service Definition:
  ``` yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: external-service          # name of service must match name of Endpoints object
    spec:                             # no selector defined
      ports:
      - port: 80
  ```
- Endpoints Definition:
  ```yaml
    apiVersion: v1
    kind: Endpoints
    metadata:
      name: external-service      # name of Endpoints object must match name of service
    subsets:
      - addresses:
        - ip: 11.11.11.11         # ip of endpoints that the service will forward connections to
        - ip: 22.22.22.22         # can be multiple endpoints
      ports:
        - port: 80
  ```
##### Creating an alias for an external service
  - Refer to an external service by its FQDN
  - create s Service resource with the `type` field set to `ExternalName`
  ``` yaml
      apiVersion: v1
      kind: Service
      metadata:
        name: external-service
      spec:
        type: ExternalName                         1
        externalName: someapi.somecompany.com      2
        ports:
        - port: 80
  ```
  - pods can connect to the external service through `external-service.default.svc.cluster.local` domain name instead of using the service's actual FQDN
  - hides actual service name and its location from pods consuming service, which allows modification of the service FQDN by changing `spec.externalName`
  - `ExternalName` services are implemented solely at the DNS level - a simple CNAME DNS record is created for the service, so these type of services don't get a cluster ip
      - CNAME record points to a FQDN instead of a numberic IP address
  - By default external endpoints eg `www.google.com` are reachable from within the cluster, the `external-service` Service just enables decoupling

#### Exposing Services to External clients
- Expose certain services, such as a frontend webservers to the outside
![exposing service to clients outisde cluster][fig_5_5]
- Ways to make service accessible externally:
  - Set service type to `NodePort`
  - Set service type to `LoadBalancer`
  - Create `Ingress` resource
      - a radically different mechanism for exposing multiple services through a single IP address
      - Operates at HTTP level

##### NodePort
- Each cluster node opens a port on the **node** itself, redirects traffic received on that port to the underlying service
- The service is accessible through the internal cluster IP and port AND the dedicated port on **all** nodes
  (able to connect via any node's ip addresses and reserved node port)
``` yaml
  spec: 
    type: NodePort
    ports:
    - port: 80          # port to service's internal cluster IP
    targetPort: 8080    # target port of backing pods
    nodePort: 30123     # Service will be accessible through port 30123 of each of the cluster nodes
```
  - Incoming connection to one of the nodes ports (<node_ip>:30123) will be redirected to a randomly selected pod, 
    which may or may not be the one running on the node the connection is being made to
  ![Request routing for nodePort service][fig_5_6]
  - Issue: When node fails, client cannot access service anymore...hence using an external load balencer is more practical

##### LoadBalancer
- Extension of the `NodePort` type, makes the service accessible through a dedicated load balancer, provisioned from the cloud infrastructure that K8s is running on
- Load balancer redirects traffic to the node port across all nodes
- Clients connect to the service through load balancer's unique, publicly accessible IP address
- Note: Relies on environment to support `LoadBalancer` services, if unsupported, will default to `NodePort` service
- Note: Session affinity & Web browsers: 
  - Even when session affinity is set to None, requests sent via `web browser` will hit the exact same pod every time. This is because:
    - Browsers use `keep-alive` connections, sending all its requests through a single connection
    - `curl` opens a new connection every time
    - Services work at the connection level..so when a connection to a service is first openend, 
      a random pod is selected and then all network packets belonging to that connection are all 
      sent to that single pod (Users will always hit the same pod until the connection is closed)
![load_balancer type service][fig_5_7]

##### Things to note for NodePort and LoadBalancers
- when external client connects to a service through a node port, the randomly 
  chosen pod may / may not be running on the same node that received the connection
- additional network hop required to reach pod may not always be desirable
- adding `spec.externalTrafficPolicy: Local` restricts the service proxy to choose a 
  locally running pod
  - if no local pod exists, the connection will hang (won't be forwarded to random
    global pod)
  - need to ensure load balancer forwards connections only to nodes that have at
    least 1 such pod
  - Can result in uneven distribution of requests
  ![uneven distribution of requests to pods][fig_5_8]
  - Does not perform SNAT, so actual client's IP is preserved, backing pod can see 
    client's IP address (because there is no additional hop)

##### Exposing services externally through Ingress
![exposing multiple services through single ingress][fig_5_9]
![Accessing pods through an Ingress][fig_5_10]
- Each `LoadBalancer` service requires its own load balancer with its own public IP address,
  an Ingress only requires **one** but can support many services
- Ingress examines host and path in HTTP request to determine which service request should be forwarded to
- Ingersses operate at the application layer of the network stack, can provide features such as cookie-based session affinity (which services can't)
- requires an `Ingress Controller` running in the cluster for Ingress resources to work (differs among various K8s environments)
- Important to ensure domain name resolves to IP of the **Ingress Controller**
  - configure DNS servers to resorve *domain name* to Ingress Controller IP

###### Ingress Definition
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia-ingress
spec:
  rules:
  - host: kubia.example.com     # Ingress maps kubia.example.com domain name to your service
    http:
      path:
      - paths: /                # all incoming HTTP requests for host kubia.example.com will be sent to
        backend:                # kubia-nodeport service at port 80
          serviceName: kubia-nodeport
          servicePort: 80
      - path: /admin
        backend:
          serviceName: kubia-admin
          sevicePort: 80
  - host: bar.example.com       # map to service via host
    http:
      paths: 
        - path: /
          backend:
            serviceName: bar
            servicePort: 80
```

###### Configuring Ingress to handle TLS Traffic
- By attaching a certificate and private key to the Ingress (Enables Ingress Controller to take care of everything related to TLS)
    - Cert and Private Key need to be stored in a `Secret` resource
        ```
        $ openssl genrsa -out tls.key 2048
        $ openssl req -new -x509 -key tls.key -out tls.cert -days 360 -subj
         /CN=kubia.example.com
        $ kubectl create secret tls tls-secret --cert=tls.cert --key=tls.key
        secret "tls-secret" created
        ```
    - Sign cert through `CertificateSigningRequest` resource (signer component must be running in a cluster)
- Add a `spec.tls` component into Ingress Manifest
```yaml
  tls:              # whole tls configuration under tls
  - hosts:                
    - kubia.example.com
    secretName: tls-secret
  rules:
    ....
```


#### Readiness Probes
- Readiness Probe give time for a pod to do its initializing procedure, before being added into the `Endpoints` object to serve incoming requests
- Likewise for Pods that fail readiness probe:
![Pod that fails readiness probe][fig_5_11]


#### Headless Service
- Used to connect to ALL pods behind a service instead of just 1
- setting `clusterIP` field in a service spec to `None` makes the service headless
- K8s won't assign the service a cluster IP
- Enables discovery of pod IPs through DNS 


[fig_5_1]: ./images/05fig01_alt.jpg
[fig_5_3]: ./images/05fig03_alt.jpg "Figure 5.3, Using kubectl exec to test out a connection to the service by running curl in one of the pods"
[fig_5_4]: ./images/05fig04_alt.jpg "Figure 5.4, Pods consuming a service with 2 external endpoints"
[fig_5_5]: ./images/05fig05_alt.jpg "Figure 5.5, Exposing service to external clients"
[fig_5_6]: ./images/05fig06_alt.jpg "Figure 5.6, an external client connecting to a NodePort service either through Node 1 or 2"
[fig_5_7]: ./images/05fig07_alt.jpg "Figure 5.7. An external client connecting to a LoadBalancer Service"
[fig_5_8]: ./images/05fig08_alt.jpg "Service using Local external traffic policy may lead to uneven load distribution across pods"
[fig_5_9]: ./images/5fig09_alt.jpg "Figure 5.9 Multiple services can be exposed through a single ingress"
[fig_5_10]: ./images/05fig10_alt.jpg "Figure 5.10 Accessing pods through an Ingress"
[fig_5_11]: ./images/05fig11_alt.jpg "Figure 5.11 Pod whose readiness probe fails is removed as an endpoint of a service"
