# Lab: Deploying Apps in Kubernetes

## Overview
In this Lab we will explore how the Kubernetes resources such as Deployments
and Services can enable us to make application deployments using best practice
patterns to reduce downtime, test new features, and easily roll back to previous
releases.

## Lab Format
Within code snippets, lines beginning with a bash prompt (`$`) denote commands
you should enter. Do not type the `$` symbol. For example:

```console
$ kubectl -h
```

Lines that do NOT begin with a `$` denote example console output you may see
when you type the previous command. You do not need to enter these lines.

## Task 1: Test the Recreate strategy

### Step 1: Examine the Deployment strategy property

```console
$ kubectl explain deployments.spec.strategy
```

### Step 2: Create and apply a Deployment

Using the following YAML, create a deployment manifest file and apply it to the
cluster with `kubectl`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
  labels:
    app: http-app
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: http-app
  template:
    metadata:
      labels:
        app: http-app
    spec:
      containers:
      - name: http-app
        image: jrrickerson/http-app:v1
        ports:
        - containerPort: 80
```

### Step 3: Create and apply a LoadBalancer type Service

Using the following YAML, create and apply a Service manifest to route traffic to our Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: http-app
  name: app-service
spec:
  ports:
  - port: 8888
    protocol: TCP
    targetPort: 80
  selector:
    app: http-app
  sessionAffinity: None
  type: LoadBalancer
```

### Step 4: Run a script to test traffic on your Service

Run the following script in a separate Bash / zsh shell

```bash
while true; do date +%T; curl localhost:8888; sleep 2; done
```

### Step 5: Perform an upgrade of our application

Using your editor, modify the Deployment manifest file. Change the image within
the Pod spec section from `jrrickerson/http-app:v1` to `jrrickerson/http-app:v2`
to simulate a new release of our application. Reapply the new manifest edits.

### Step 6: Watch the update

Using the command `kubectl get pods --watch` to continuously display the list
of Pods. Note how the Recreate strategy tears down all existing v1 Pods before
starting up the v2 Pods. You may also see some failed requests in your separate
shell window.

## Task 2: Switch to a Rolling Update strategy

### Step 1: Examine RollingUpdate options

```console
$ kubectl explain deployments.spec.strategy.rollingUpdate
```

### Step 2: Create and apply a new Deployment manfest

Use the following YAML to reconfigure the Deployment. Note the Rolling Update settings:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
  labels:
    app: http-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 0
  selector:
    matchLabels:
      app: http-app
  template:
    metadata:
      labels:
        app: http-app
    spec:
      containers:
      - name: http-app
        image: jrrickerson/http-app:v1
        ports:
        - containerPort: 80
```

### Step 3: Simulate an Application upgrade

Change the Deployment manifest file image from `jrrickerson/http-app:v1` to
`jrrickerson/http-app:v2` and repply to perform the simulated upgrade.

### Step 4: Observe the upgrade in progress

Using `kubectl get replicaset -o wide` examine the new ReplicaSet the
Deployment creates for the new v2 Pods as it drains the existing
ReplicaSet of v1 Pods.

Note also that in your separate shell window making continuous requests,
you should not see any failed requests, but you will temporarily see a
mix of "v1" and "v2" responses.

### Step 5: Perform an automatic rollback

Occasionally a release of a new version may have unforseen issues. Kubernetes
Deployments can make it simple to roll back to the previous working version.

Execute the command `kubectl rollout undo deployment app-deployment` to roll
back the upgrade to "v2" and go back to the previous "v1" image.

Using the command `kubectl get replicaset -o wide`, observe the rollback,
which essentially just reverses the upgrade process.

## Task 3: Clean Up

Use Ctrl-C to stop the infinite loop of `curl` requests in your shell window.

```console
$ kubectl delete service app-service
```

```console
$ kubectl delete deploy app-deployment
```

## Task 4: Canary Deployment

In a Canary deployment, we usually want only *some* portion of our traffic to
be routed to the new version of application so we can test and ensure stability
before putting all of the normal traffic load on the application.

This is usually somewhat more manual than other deployment strategies, though
it can be automated with a robust set of smoke tests and load tests.

### Step 1: Create the v1 Deployment

#### Create the v1 manifest

Create and apply a Deployment manifest, based on the following YAML.
Name the file something like `deployment_v1.yaml` as you will have multiple of these.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment-v1
  labels:
    app: http-app
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: http-app
  template:
    metadata:
      labels:
        app: http-app
        version: v1
    spec:
      containers:
      - name: http-app
        image: jrrickerson/http-app:v1
        ports:
        - containerPort: 80
```

Note this manifest specifies the extra label `version: v1` in the Pod Template.
Labels can have as many key-value pairs as you like.

#### Recreate the Service object

Re-apply the Service manifest you created earlier to expose the Deployment
Pods behind a LoadBalancer.

All 3 Pods from your v1 Deployment should now be serving traffic. Since the
`selector` property on the Service ONLY specifies `app: http-app`, all 3 Pods
match the pattern. The `version: v1` label is effectively ignored.

### Step 2: Create the v2 "canary"

#### Create the second Deployment

Create a *completely separate* deployment for version 2 of our application,
leaving the existing version 1 in place and serving traffic.

To do this, first create a new v2 Deployment manifest and apply it, based on
the YAML provided below. Make sure to save this as a separate file
(ex. `deployment_v2.yaml`) from the v1 Deployment.

Note the difference in labels and container image tags!

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment-v2
  labels:
    app: http-app
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: http-app
  template:
    metadata:
      labels:
        app: http-app
        version: v2
    spec:
      containers:
      - name: http-app
        image: jrrickerson/http-app:v2
        ports:
        - containerPort: 80
```

#### Test the Service

Make a few requests with your browser or `curl` to localhost:8888 and examine
the responses carefully. If you do this several times, you should notice that
your v1 Pods are serving ~75% of the requests, and the v2 "canary" Pod is
serving the remaining 25%. This is because the Service selector currently
matches ALL FOUR Pods (3 v1, 1 v2) and splits the traffic evenly between Pods.

Using this same method with different numbers of Pods, we can effectively
serve any percentage of traffic to our canary that we like, and monitor the
results.

#### Set up a new Service for testing

Often in Canary deployments, we may want to set up a means for developers,
beta testers, QA personnel, or whoever to access ONLY the new version
for testing purposes. We can do that through another Service definition.

Create a new Service that serves ONLY v2 traffic based on this YAML. Save it
as a separate manifest file from the initial Service

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: http-app
  name: app-service-v2
spec:
  ports:
  - port: 8889
    protocol: TCP
    targetPort: 80
  selector:
    app: http-app
    version: v2
  sessionAffinity: None
  type: LoadBalancer
```

Note that this new Service has a different name, exposes a different port
(since we cannot share ports on localhost), and an additional `selector`
property, `version: v2`.

Since ALL the selectors on a Service must match for the traffic to be routed
to the Pods, only the single v2 "canary" Pod can serve requests to this new
Service.

#### Test the new Service

Using your browser or `curl`, make several requests to http://localhost:8889,
and you should see ONLY responses from the new v2 Pod.

### Step 3: Finish the v2 rollout

If all the canary testing was successful, it's now time to finish rolling out
v2 of the application. We can do this easily and gradually using the scaling
feature of the Deployment resource.

#### Scale up the v2 Deployment

Edit your v2 Deployment manifest and set the number of replicas to 3, and
reapply it.

The v1 and v2 Deployments are now serving 50% of the traffic each.

#### Scale down the v1 Deployment

Edit your v1 Deployment manifest and set the number of replicas to 1, and
reapply it.

The v2 Deployment now handles 75% of the traffic, and the v1 is only serving
25% of the traffic.

Note that if there were any stability issues with v2 at this point, you could
simply re-scale each deployment to "undo" the rollout.

#### Delete the v1 Deployment

Using `kubectl delete`, delete the v1 Deployment entirely to complete the
rollout of v2.

#### Test the original Service

If you make requests to the original service on localhost:8888, you should
now only ever see v2 responses.

## Task 5: Cleanup

Delete the resources you created in this lab.

```console
$ kubectl delete service app-service-v2
```

```console
$ kubectl delete service app-service
```

```console
$ kubectl delete deploy app-deployment-v2
```
