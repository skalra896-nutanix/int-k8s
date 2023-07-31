# Lab: Kubernetes Cluster Basics

## Overview
In this lab, we will use the `kubectl` command to interact with the Kubernetes
API in order to run some simple workloads. You will explore additional
capabilities of the `kubectl` tool and gain familiarity with it.

## Lab Format
Within code snippets, lines beginning with a bash prompt (`$`) denote commands
you should enter. Do not type the `$` symbol. For example:

```console
$ kubectl -h
```

Lines that do NOT begin with a `$` denote example console output you may see
when you type the previous command. You do not need to enter these lines.

## Task 1: Run a single pod

### Step 1: Explore the `kubectl` tool

#### Examine the `kubectl` subcommands:

```console
$ kubectl -h
kubectl controls the Kubernetes cluster manager.

 Find more information at: https://kubernetes.io/docs/reference/kubectl/

Basic Commands (Beginner):
  create          Create a resource from a file or from stdin
  expose          Take a replication controller, service, deployment or pod and
expose it as a new Kubernetes service
  run             Run a particular image on the cluster
  set             Set specific features on objects
...
```

We will be using several of these commands throughout our labs.

#### View the global options for `kubectl`:

```console
$ kubectl options
The following options can be passed to any command:

    --as='':
        Username to impersonate for the operation. User could be a regular
        user or a service account in a namespace.

    --as-group=[]:
        Group to impersonate for the operation, this flag can be repeated to
        specify multiple groups.

    --as-uid='':
        UID to impersonate for the operation.
...
```

You can also use the `kubectl` cheatsheet to read about the various subcommands
and options: https://kubernetes.io/docs/reference/kubectl/cheatsheet/

#### List Kubernetes resources

```console
$ kubectl get all
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.43.0.1    <none>        443/TCP   6d16h
```

Since we haven't created any resources yet, the only resources you should see
are the control plane resources Kubernetes itself uses. Note that if you have
more restricted permissions to a cluster, you may not see these resources.

### Step 2: Run a single Pod

Using the `kubectl run` command, we can run a single Pod from an image, similar
to how you would run a container using `docker run` or `nerdctl run`, except
that the cluster decides *where* to run the Pod on the cluster for us.

#### Examine the `kubectl run` command

```console
$ kubectl run --help
```

#### Run a single Pod of "nginx"

```console
$ kubectl run --image=nginx nginx
pod/nginx created
```

#### View the newly created Pod

```console
$ kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          46s
```

#### Get detailed information about the Pod

```console
$ kubectl get pods -o wide
NAME    READY   STATUS    RESTARTS   AGE    IP           NODE                   NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          3m1s   10.42.0.14   lima-rancher-desktop   <none>           <none>
```


#### Get additional information with "describe"

```console
$ kubectl describe pod nginx
Name:             nginx
Namespace:        default
Priority:         0
Service Account:  default
Node:             lima-rancher-desktop/192.168.5.15
Start Time:       Mon, 31 Jul 2023 07:55:01 -0400
Labels:           run=nginx
Annotations:      <none>
Status:           Running
IP:               10.42.0.14

...

Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  13m   default-scheduler  Successfully assigned default/nginx to lima-rancher-desktop
  Normal  Pulling    13m   kubelet            Pulling image "nginx"
  Normal  Pulled     13m   kubelet            Successfully pulled image "nginx" in 18.733684591s (18.733715925s including waiting)
  Normal  Created    13m   kubelet            Created container nginx
  Normal  Started    13m   kubelet            Started container nginx
```

Note the Events section at the bottom of the output of `kubectl describe`. This
allows you to see the actions the cluster has taken to start running your
workload, and can be a useful tool in diagnosing issues!

#### Make a request to the Pod

Currently, our Pod has an IP address, but it is only availble INSIDE the
cluster (i.e. to other Pods), so we cannot access it directly. We can use
`kubectl port-forward` to temporarily forward a local port to the Pod for testing.

```console
$ kubectl port-forward pods/nginx 8888:80
Forwarding from 127.0.0.1:8888 -> 80
Forwarding from [::1]:8888 -> 80
```

Now, direct your browser or use a tool like `curl` to make a request to 
http://localhost:8888 and you should see the output from our nginx container!

You can use CTRL-C to cancel the port forwarding when you have finished.

#### View the container logs to see your requests

```console
$ kubectl logs nginx
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
...
127.0.0.1 - - [31/Jul/2023:12:24:37 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.81.0" "-"
```

#### Clean up your standalone pod

```console
$ kubectl delete pod nginx
pod "nginx" deleted
```

## Task 2: Scale an application

Using a cluster to run a single Pod does provide an advantage over a single
host if we have multiple nodes but the real power in Kubernetes comes from
the ease of scalability. We'll briefly demonstrate the basics of scaling Pods
via Deployments.

### Step 1: Create a Deployment to manage our Pods

```console
$ kubectl create deployment --image=nginx --replicas=3 nginx
deployment.apps/nginx created
```

#### Examine our Deployment and Pods

```console
$ kubectl get deployment
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   3/3     3            3           7s
```

```console
$ kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE    IP           NODE                   NOMINATED NODE   READINESS GATES
nginx-77b4fdf86c-kfvvh   1/1     Running   0          115s   10.42.0.15   lima-rancher-desktop   <none>           <none>
nginx-77b4fdf86c-jjgm4   1/1     Running   0          115s   10.42.0.16   lima-rancher-desktop   <none>           <none>
nginx-77b4fdf86c-5xrz5   1/1     Running   0          115s   10.42.0.17   lima-rancher-desktop   <none>           <none>
```

These Pods are all running on the same node in my local cluster setup, but in a
production cluster with multiple nodes, it is far more likely for the control
plane to distribute our workload to multiple nodes. Note that the IPs of each
Pod will be in the Pod IP subnet regardless of what Node they are scheduled on.

#### View Pod resource usage

```console
$ kubectl top pod
NAME                     CPU(cores)   MEMORY(bytes)   
nginx-77b4fdf86c-5xrz5   0m           2Mi             
nginx-77b4fdf86c-jjgm4   0m           2Mi             
nginx-77b4fdf86c-kfvvh   0m           2Mi
```

### Step 2: Scale the Deployment

#### Scale up by creating more pods

```console
$ kubectl scale deployment nginx --replicas=5
deployment.apps/nginx scaled
```

```console
$ kubectl get pods
```

#### Scale back down

```console
$ kubectl scale deployment nginx --replicas=2
deployment.apps/nginx scaled
```

```console
$ kubectl get pods
```

## Task 3: Explore `kubectl` command

Try a few of the following commands and examine their output. Look in the
documentation for details on how these work.

```console
$ kubectl get events
```

For these commands, replace `<pod name>` with one of the names of the Pods from
your previous outputs.

```console
$ kubectl exec <pod name> -- ls -al
```

```console
$ kubectl exec -it <pod name> -- /bin/bash
```
NOTE: Type `exit` to exit the bash shell

## Task 4: Clean up

```console
$ kubectl delete deployment nginx
```
