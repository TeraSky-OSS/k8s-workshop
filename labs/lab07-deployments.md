# Lab 7: Deployments and YAML

A Deployment declares your desired state: which container image to run, how many replicas, and how to roll out updates. Kubernetes continuously reconciles the live state toward that desired state. Underneath, a Deployment manages a ReplicaSet, which is the object that actually maintains the right number of running Pods.

This lab covers the full Deployment lifecycle: create, inspect, scale, update, roll back, and edit YAML directly.
---

## Step 1: Create a Deployment

Create an nginx Deployment:

```console
kubectl create deployment nginx --image=nginx
```

Example output:

```console
deployment.apps/nginx created
```

List the Deployment. The `READY` column shows `available/desired`:

```console
kubectl get deployment nginx
```

Example output:

```console
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           5s
```

Get a detailed breakdown including the update strategy, selector, and recent events:

```console
kubectl describe deployment nginx
```

The **Events** section at the bottom shows that the Deployment created a ReplicaSet.

List the ReplicaSet the Deployment created. Its name includes the Deployment name and a hash of the Pod template:

```console
kubectl get replicaset
```

Example output:

```console
NAME               DESIRED   CURRENT   READY   AGE
nginx-6d4cf56db6   1         1         1       20s
```

Finally, list the Pod the ReplicaSet created. Notice the Pod name inherits the ReplicaSet name with another hash appended:

```console
kubectl get pods
```

Example output:

```console
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6d4cf56db6-xz8tk   1/1     Running   0          25s
```

The naming chain `deployment -> replicaset -> pod` is how Kubernetes tracks ownership. If you delete a Pod here, the ReplicaSet immediately creates a replacement.

Get the full YAML of the live Deployment to understand its structure:

```console
kubectl get deployment nginx -o yaml
```


---

## Step 2: Generate YAML without creating anything

`--dry-run=client -o yaml` runs the command locally, builds the object it would send to the API server, and prints it as YAML - without creating anything in the cluster. It is a quick way to bootstrap a manifest for any resource.

Generate a Deployment manifest for 5 nginx replicas:

```console
kubectl create deployment nginx --image=nginx --replicas=5 --dry-run=client -o yaml
```

Example output:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 5
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

The output is valid Kubernetes YAML. Redirect it to a file to use as a starting point:

```console
kubectl create deployment nginx --image=nginx --replicas=5 --dry-run=client -o yaml > nginx-deployment.yaml
```

This works for most `kubectl create` subcommands - Deployments, Services, ConfigMaps, and more. Use it whenever you need a YAML scaffold rather than writing one from scratch.

---

## Step 3: Scale the Deployment

Scaling changes the `replicas` field in the Deployment spec. Kubernetes detects the gap and creates new Pods to fill it.

Scale to 3 replicas:

```console
kubectl scale deployment nginx --replicas=3
```

Watch the new Pods start. The `-w` flag streams updates until you press Ctrl+C:

```console
kubectl get pods -w
```

You will see two new Pods appear with `ContainerCreating` and then transition to `Running`. Press Ctrl+C to stop watching.

---

## Step 4: Update the image

Updating the image triggers a rolling update. The Deployment creates a new ReplicaSet for the new version and gradually shifts Pods from the old ReplicaSet to the new one. At no point does traffic hit zero running Pods.

Update the nginx container to version 1.30:

```console
kubectl set image deployment/nginx nginx=nginx:1.30
```

Example output:

```console
deployment.apps/nginx image updated
```

Watch the rolling update progress:

```console
kubectl rollout status deployment/nginx
```

Example output:

```console
Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
deployment "nginx" successfully rolled out
```

After the rollout completes, check the ReplicaSets. You will now see two: the old one (scaled to 0) and the new one (scaled to 3). The old one is kept so a rollback is instant:

```console
kubectl get replicaset
```

---

## Step 5: Roll back to revision 1

Kubernetes keeps a history of each configuration change to a Deployment. Check it:

```console
kubectl rollout history deployment/nginx
```

Example output:

```console
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

Revision 1 is the original nginx image. Revision 2 is the `nginx:1.30` update. Rolling back just reactivates the old ReplicaSet (the one already sitting at 0 replicas):

```console
kubectl rollout undo deployment/nginx
```

Example output:

```console
deployment.apps/nginx rolled back
```

Confirm the image reverted to the original nginx tag:

```console
kubectl describe deployment nginx | grep Image
```

The rollback was instant because the old ReplicaSet was never deleted, only scaled down.

---

## Step 6: Edit YAML and apply

So far you've used imperative commands. The declarative approach exports the object as YAML, edits it, and applies the result. `kubectl apply` computes the difference between what you send and what exists in the cluster and only changes what differs.

Export the Deployment YAML to a file:

```console
kubectl get deployment nginx -o yaml > nginx-deployment.yaml
```

Open the file in nano:

```console
nano nginx-deployment.yaml
```

Use Ctrl+W to search for `replicas`. Change `replicas: 3` to `replicas: 1`.

Save the file: press Ctrl+O, then Enter. Exit nano: press Ctrl+X.

Apply the updated file:

```console
kubectl apply -f nginx-deployment.yaml
```

You may see a warning about a missing `last-applied-configuration` annotation. This is expected when applying a file that was exported with `kubectl get` rather than created declaratively. It can be safely ignored - kubectl will patch the annotation automatically.

Watch the Pod count drop from 3 to 1. Two Pods will be terminated:

```console
kubectl get pods
```

---

## Step 7: Use kubectl edit to revert

`kubectl edit` is a shortcut that fetches the live object, opens it in an editor, and applies your changes when you save. It operates directly against the API without needing a local file.

Open the live Deployment in nano:

```console
EDITOR=nano kubectl edit deployment nginx
```

Use Ctrl+W to search for `replicas`. Change `replicas: 1` back to `replicas: 3`.

Save and exit: press Ctrl+O, then Enter, then Ctrl+X.

Kubernetes applies the change immediately. Watch Pods scale back up:

```console
kubectl get pods -w
```

Press Ctrl+C to stop watching once all three Pods are Running.

---

## Cleanup

Delete the Deployment. This also deletes the ReplicaSets and Pods it owns:

```console
kubectl delete deployment nginx
```

Example output:

```console
deployment.apps "nginx" deleted
```

Confirm all Pods are gone:

```console
kubectl get pods
```

---
