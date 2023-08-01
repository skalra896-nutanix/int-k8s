# Lab: Using Data Persistence

## Overview
In this lab we will explore how the Kubernetes resources StorageClass,
PersistentVolume, and PersistentVolumeClaim work together to allow our
otherwise ephemeral Pod workloads to store and access data, regardless of
where they are scheduled in the cluster.

## Lab Format
Within code snippets, lines beginning with a bash prompt (`$`) denote commands
you should enter. Do not type the `$` symbol. For example:

```console
$ kubectl -h
```

Lines that do NOT begin with a `$` denote example console output you may see
when you type the previous command. You do not need to enter these lines.

## Task 1: Dynamic Provisioning

While PersistentVolumes can be statically provisioned in Kubernetes, it is
often a cumbersome task. Cluster administrators must pre-provision the volumes
on the storage provider, and the PersistentVolumeClaim can only make a
best-effort selection from the list of available PersistentVolumes.

Generally, statically provisioned PVs are best used for large, read-only data
that may need to be shared amongst many workloads, but don't change often,
and have predicatble storage sizes. Consider a library of shared images and
videos for a CMS, or a large set of training data for a machine-learning
model.

More frequently, Dynamic provisioning is used. A PVC is created to request
a specific amount of storage for a workload, and that requested is fulfilled
on-demand by a storage provider, configured via a StorageClass. This is
generally performed in cloud-based clusters by virtual disk platforms like
AWS's Elastic Block Store or Google Cloud's Persistent Disks.

### Step 1: Examine the default StorageClass

Our local cluster comes with a StorageClass called `local-path`, fulfilled
by our local host's filesystem. You can get additional details:

```console
$ kubectl describe sc local-path
```

### Step 2: Create a PersistentVolumeClaim resource

A PVC tells the cluster that we want a volume of storage available to use
by our Pods. The underlying storage provider may not *actually* provision
the storage until it is in use, however.

Create a PVC manifest using the following YAML and apply it.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: html-files
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Note that we did not specify a StorageClass here, so the default is used.
You may see the PVC status as "Pending." This is because the StorageClass
defines the property `VolumeBindingMode: WaitForFirstConsumer`, which means
it will not actually create a volume until a Pod attempts to use the PVC.

### Step 3: Create Pods to read the PVC

#### Create a Deployment manifest

Using what you've learned and the documentation, create a Deployment manifest
with the following criteria:

* Replica count: 2
* Image: nginx
* Pod Spec mounts the PVC as a volume using the YAML snippets below

```yaml
# Container Volume Mount
volumeMounts:
  - mountPath: "/usr/share/nginx/html"
    name: html-files
```

```yaml
# Pod Volume definition
volumes:
  - name: html-files
    persistentVolumeClaim:
      claimName: html-files
      readOnly: true
```

#### Apply and test the Deployment

Apply your Deployment manifest, and inspect it and the Pods it creates using
`kubectl`. You should see your Pods have a Mount defined with the read-only
flag (ro) and the path your specified, and your PVC should now show a status
of "BOUND" (Note this may take up to a minute). Additionally, you should be
able to use `kubectl get pv` to see the volume that was created and get
additional information about it.

Create and apply a Service manifest with a selector that matches your
Deployment Pods' labels and make a request to the service. You should see
the default nginx "Welcome" message.


### Step 4: Create a Pod to write to the PVC

#### Create a one-off Pod manifest

Using the YAML below, create a Pod manifest and apply it. Note the differences
from previous Pods - here we have set the restartPolicy to "Never" because
we only want the Pod to execute a single command and quit. This Pod will also
mount our same PVC, but in read-write access mode by default.

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: html-downloader
spec:
  containers:
    - name: busybox
      image: busybox
      workingDir: "/data"
      command: ["wget"]
      args: 
        - "https://github.com/jrrickerson/int-k8s/blob/main/web/index.html"
        - "-O"
        - "index.html"
      volumeMounts:
      - mountPath: "/data"
        name: html-files
  volumes:
    - name: html-files
      persistentVolumeClaim:
        claimName: html-files
```


### Step 5: Test your new HTML file

Once your "downloader" Pod has finished downloading (this should only take a
couple of seconds), retry your requests to your Service. You should now see the
contents of the `index.html` I provided for you, which was written to the PV
by the downloader and read by the nginx Pods.

You can try scaling up or down your Deployment to see that all the Pods
continue to share the same data without having to copy it or redownload
anything.

### Step 6: Cleanup

Using `kubectl`, delete your Deployment, one-off downloader Pod, and PVC
resources. The PV will be automatically cleaned up once the PVC is deleted due
to the ReclaimPolicy of `Delete` on the StorageClass.


## Bonus Task: Configure BasicAuth for nginx

For a bonus challenge, we will use a ConfigMap and a Secret object to configure
an nginx workload and provide a username and password to secure a website.

Please Note: This exercise is for demonstration purposes ONLY. Basic Auth is
NOT a recommended way of securing a web application.

### Step 1: Create a ConfigMap

The nginx webserver / reverse proxy uses a config file called `default.conf`,
which it expects to read from a standard location. A typical configuration
looks something like this:

```nginx
server {
      listen       80;
      server_name  localhost;
      location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        auth_basic           "closed site";
        auth_basic_user_file "/etc/nginx/htpasswd";
      }
}
```

#### Create the ConfigMap manifest

Using the nginx configuration above, and the YAML syntax of a `|` character
for a multi-line string value, create a ConfigMap manifest with a key called
`default.conf` with the value the nginx configuration.

Apply the ConfigMap manifest.

### Step 2: Create a Secret

I have pre-hashed user credentials for the `htpasswd` file nginx can use for
Basic Auth. The following defines a user `user` with a password of `password1`.

```
user:$apr1$//v0t3Wj$LTIYHkRDvddkl699gPLCA/
```

Create a Secret object manfest with a key called `htpasswd` with the value of
the credentials string above. Remember that you will need to base64 encode the
string before putting it into the Secret. You can do so with the following
command:

```console
$ echo 'user:$apr1$//v0t3Wj$LTIYHkRDvddkl699gPLCA/' | base64
```

Important! Remember that base64 encoding is NOT cryptography. In a Production
cluster deployments you may need to use a more secure method for credentials,
such as a script or application that extracts credentials from an encrypted
vault and injects them into a Secret object without human interaction.

### Step 3: Create a Pod to use the configuration

Create a Pod manifest based on the YAML provided. Examine how the Pod Spec
utilizes the ConfigMap and Secret as volumes. Apply the Pod manifest to the
cluster with `kubectl`

NOTE: Make sure the name you chose for the ConfigMap and Secret match the
name used in the Pod spec!

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: web-server-pod
spec:
  containers:
    - name: frontend
      image: nginx
      volumeMounts:
      - mountPath: /etc/nginx/conf.d/default.conf
        name: config-volume
        subPath: default.conf
      - mountPath: "/etc/nginx/htpasswd"
        name: creds-volume
        subPath: htpasswd
  volumes:
    - name: creds-volume
      secret:
        secretName: nginx-creds
    - name: config-volume
      configMap:
        name: nginx-config
```

### Step 4: Test the configuration

#### Make a standard web request

Using `kubectl port-forward`, forward a local port to the Pod and make a
browser or `curl` request to the forwarded port. You should get an error,
`401 Authorization Required` instead of the normal HTML response.

#### Make an authorized web request

Use the following command to make an authorized web request and get a
successful response.

```console
$ curl --basic --user user:password1 localhost:<forwarded port>
```

### Step 5: Cleanup

```console
$ kubectl delete pod web-server-pod
```

```console
$ kubectl delete secret nginx-creds
```

```console
$ kubectl delete configmap nginx-config
```
