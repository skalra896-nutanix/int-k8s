# Lab: Declarative Kubernetes Resources

## Overview
In this lab we will create Kubernetes API resources declaratively by authoring
YAML manifest files, which can then be sent to the API to be applied to the
cluster. As plain text, these files can be stored in source control,
automatically applied by CI/CD pipelines, and re-applied periodically to
avoid configuration drift.

## Lab Format
Within code snippets, lines beginning with a bash prompt (`$`) denote commands
you should enter. Do not type the `$` symbol. For example:

```console
$ kubectl -h
```

Lines that do NOT begin with a `$` denote example console output you may see
when you type the previous command. You do not need to enter these lines.

## Task 1: Understand YAML manifests

Kubernetes manifest files can be written in JSON or YAML, but YAML tends to be
a little more human-friendly. If you aren't familiar with YAML, there's a good
basic guide to the syntax here:
https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html

Note that in YAML, the indentation is *required* to be consistent. Many popular
IDEs and editors support YAML and will do syntax checks for you.

The manifest allows us to declare the state we want the cluster to be in, and
allow the control plane components to figure out how to get there. This means
we don't have to care (usually) about the CURRENT state, just the one we want
to have at the end.

### Step 1: Review a Deployment

#### Examine the YAML manifest

Examine the following Deployment defintion, and see if you can answer the
following questions:
- What will be the name of the resource?
- What tells you what type of resource it is?
- How many Pods will be created when this Deployment is applied?
- Which container image will the Pods be created with?

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

Note also what the manifest example DOES NOT declare - IP addresses, which Node
to run the Pod on, etc. That's the control plane's responsibility, and we
shouldn't generally have to care about those details.

#### Get information on the Deployment resource defintion

The Kubernetes API itself can tell you about the definition of a Deployment in
YAML. Try the following commands to see what properties are available on a Deployment:

```console
$ kubectl explain deployment
```

```console
$ kubectl explain deployment.spec
```

```console
$ kubectl explain deployment.spec.replicas
```

NOTE: You can also refer to the Kubernetes Reference documentation, which is
generated from the code itself: https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/deployment-v1/

### Step 2: Create a Deployment

#### Create a Deployment manifest file

Using your favorite editor, create a file called `deployment.yaml` with the
following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: http-deployment
  labels:
    app: http
spec:
  replicas: 3
  selector:
    matchLabels:
      app: http
  template:
    metadata:
      labels:
        app: http
    spec:
      containers:
      - name: http-server
        image: jrrickerson/http-app
        ports:
        - containerPort: 80
```

#### Apply the deployment

```console
$ kubectl apply -f deployment.yaml
```

This should create your Deployment and start the appropriate number of Pods as
you requested in the cluster.

#### Verify your results

Using `kubectl`, issue commands to ensure the Deployment was created
successfully. If you dont see what you expect to see, try diagnosing the
issue with `kubectl describe` or `kubectl get` commands.

### Step 3: Test Deployment autohealing

#### Delete an existing Pod
Choose the name of one Pod as part of your Deployment and delete it with:

```console
$ kubectl delete pod <pod_name>
```

#### Verify the deployment state
Using `kubectl`, verify the Pod you specified is no longer present, and that
the Deployment has created a new Pod to replace it and get the number of
replicas back to the state you requested.


## Task 2: Create a Service object

### Step 1: Apply a Service manifest

#### Review the Service YAML definition

```console
$ kubectl explain service
```

#### Write a Service manifest

Using your favorite editor, write a file called `service.yaml` with the
following content:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: http-service
spec:
  type: LoadBalancer
  selector:
    app: http
  ports:
  - protocol: TCP
    port: 8888
    targetPort: 80
```

#### Apply the Service manifest

```console
$ kubectl apply -f service.yaml
```

### Step 2: Test the service and verify Round Robin behavior

The "Traefik" component installed in our local clusters automatically forwards
requests to ports above 1024 into our cluster Service IP address block due to
the property `type: LoadBalancer`. On a Production cloud-based cluster, this
would create a cloud-based Load Balancer with a real-world external IP address.

Make an HTTP request using your browser or `curl` to http://localhost:8888 to
verify the flow of traffic. If you refresh several times, you should see a
different "Host" value appear matching the Pod which handled your request.

### Step 3: Try the Session Affinity feature

Using the documentation 
(https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/#ServiceSpec),
modify your Service manifest to set the Session Affinity
property to "ClientIP". Reapply your manifest with `kubectl` after you save your
edits, and then test it again. Note that the "Host" value now stays the same for
each subsequent request.

### Step 4: Cause a Pod failure

Using `kubectl delete`, delete the specific Pod that has been handling your
requests. The Pod name should be the same as the "Host" value in the app's
output.

Try your requests again, and note that you get a different Pod name for your
Host. Your Service has made a best effort based your Session Affinity setting,
but in the event of a failure, it will still route traffic rather than fail.

### Step 5: Break the Service traffic flow

#### Edit the `selector` property of the Service

Modify your manifest and change the `selector` property to another value other
than `app: http`. Reapply your Service manifest.

#### Test the traffic

Using `curl` or your web browser, make more requests to the service and note
that you no longer see the responses from the application. This is because your
Service selectors no longer match your Pods labels - even though the Pods are
still running correctly, the Service is no longer routing traffice to them.

#### Fix the Service and test traffic again

Edit your Service manifest to fix the selector and reapply it to restore the
flow of traffic.

## Bonus Task: Add Probes to your Deployment

Occasionally, Pods may not be ready to serve traffic as soon as the process
starts, or Pods may get into a state where they can no longer service traffic
despite the process still running (for instance, they become deadlocked).
Kubernetes supports "probes" to automatically check the status of your Pods
perodically to determine if they have become "unhealthy" and need to be
restarted.

### Step 1: Add Probes to the Deployment

Using the Deployment Reference documentation, edit your `deployment.yaml` and add a
Liveness Probe and a Readiness Probe to your Deployment. Use the following
settings:

    * Liveness Probe
        * Intial Delay: 3 seconds
        * Period: 5 seconds
    * Readiness Probe
        * Initial Delay: 5 seconds
        * Period: 3 seconds


### Step 2: Test the Probes

Make a request to the app via Service using the URL http://localhost:8888/break
The app is specifically written to purposely break after that request. Using
`kubectl`, watch the Deployment to see the Probes eventually detect the failure
and restart the failed Pod. Note the traffic will continue to flow to the
remaining "healthy" Pods in the meantime, preventing any service disruption,
unless all Pods are broken within the Probe's Period setting.

## Cleanup

```console
$ kubectl delete service http-service
```

```console
$ kubectl delete deploy http-deployment
```
