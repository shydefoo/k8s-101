## Secrets & ConfigMaps

* whole idea of app configuration is to keep config options that vary between environments sepearate from app source code
* pod descriptor (kinda like app's source code) -> move config out of pod descriptor

### ConfigMaps
* map containing key/value pairs 
* contents of map are passed to containers either as **environment variables** or **files in a volume**
* env var can be refrenced in command line args using $(ENV_VAR) syntax => can pass ConfigMap entries to processes as cli args
![Pods use ConfigMaps through environment variables and configMap volumes][fig_7_2]
```bash
$ kubectl create configmap my-config
  --from-file=foo.json                     1 single file
  --from-file=bar=foobar.conf              2 file stored under custom key
  --from-file=config-opts/                 3 whole directory
  --from-literal=some=thing                4 literal value

```
![Creating ConfigMap from individual files, directory and literal value][fig_7_5]

#### Passing ConfigMap entry to container as environment variable
```yaml

apiVersion: v1
kind: Pod
metadata:
  name: fortune-env-from-configmap
spec:
  containers:
    - image: some-image
      env:
        - name: INTERVAL              # 1 set env var called INTERVAL
          valueFrom:                  # 2 initialize from ConfigMap key
            configMapKeyRef:          # 2
              name: fortune-config    # 3 name of ConfigMap referencing
              key: sleep-interval     # 4 set variable to whatever is stored under this key in the ConfigMap
```
![Passing ConfigMap entry as env var to container][fig_7_6]

* if ConfigMap container references does not exist when pod is created, container will not start. If ConfigMap is created after, 
  container will start without pod needing to be recreated

#### Pass all entries of a ConfigMap as env vars as once: `envFrom` attribute
    ```yaml
    spec:
      containers:
        - image: some-image
          envFrom:
          - prefix: CONFIG_   # 2 All env vars will be prefixed with CONFIG_ (optional, if omitted, env vars will have same name as keys)
            configMapRef:
              name: my-config-map

    ```
#### Use configMap volume to expose ConfigMap entries as files
* configMap volume will expose each entry of the ConfigMap as a file (file contains value)
* Process running in the container can obtain entry's value by reading the contents of the file



### Introducing Secrets
* use secrets to pass sensitive data to containers
* Secrets are also maps that hold key-value pairs
* K8s keeps Secrets safe by making sure each Secret is only distributed to the nodes that run the pods that need access to the Secret
    ```bash
    $ kubectl create secret generic fortune-https --from-file=https.key \
    --from-file=https.cert --from-file=foo
    # secret "fortune-https" created
    ```

#### Default token Secret
* Every pod has a **secret** volume attached to it automatically
```bash
$ kubectl describe secrets
Name:        default-token-cfee9
Namespace:   default
Labels:      <none>
Annotations: kubernetes.io/service-account.name=default
             kubernetes.io/service-account.uid=cc04bb39-b53f-42010af00237
Type:        kubernetes.io/service-account-token

Data
====
ca.crt:      1139 bytes                                   1 # represents everything needed to securely talk to K8s API server from within pods
namespace:   7 bytes                                      1
token:       eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...      1

```

* difference between configMap and Secret -
  * contents of Secret's entries are shown as Base64-encoded strings. ConfigMap contents are clear text
  * this allows Secret's entries to contain binary values, not only plain text (max size limited to 1MB)
* when Secret is exposed to a container through a `secret` **volume**, value of Secret entry is decoded and written to the file in its actual form

#### Using Secrets
* Mount into pod using secret volume
* Expose Secret's entry as environment variable
    ```yaml
    env:
      - name: FOO_SECRET
        valueFrom:
          secretKeyRef:             # 1 variable should be set from entry of a Secret
            name: fortune-https     # 2 name of Secret holding key
            key: foo                # key of secret to expose
    ```

#### Image Pull Secrets
* Kubernetes itself requires user to pass credentials to it (eg when pulling images from private container image registry) => done through Secrets
* referenced using `imagePullSecrets` field of pod manifest
* Eg for Docker private registry => create Secret holding credentials for Docker registry
    ```bash
    kubectl create secret docker-registry mydockerhubsecret \
    --docker-username=myusername --docker-password=mypassword \
    --docker-email=my.email@provider.com
    ```
* take note: creating a **docker-registry** Secret (not generic)
* Secret will include a single entry `.dockercfg` => same as .dockercfg file in home dir (created by Docker)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-pod
spec:
  imagePullSecrets:                 1 # enables pulling image from private registry
  - name: mydockerhubsecret         1
  containers:
  - image: username/private:tag
    name: main
```

### Best practices
* so, how to get Secrets into K8s cluster..without commiting credentials in to VCS?
* traditional manner: scp .env from local into server. init script (docker-compose, /bin/bash..wtv, can make use of this .env file)
* How to get the secrets in..? purely using procedural manner? KIV





















---
[fig_7_2]: ./images/07fig02.jpg
[fig_7_5]: ./images/07fig05_alt.jpg
[fig_7_6]: ./images/07fig06_alt.jpg
