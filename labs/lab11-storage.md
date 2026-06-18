# Lab 11: Persistent Storage

Containers are stateless by design. Their filesystem is ephemeral: everything written inside the container disappears when the container stops. Kubernetes provides several volume types to work around this, ranging from temporary scratch space to durable network-backed storage that survives Pod deletion.
---

## Step 1: EmptyDir - scratch space between containers

An `emptyDir` volume is created when a Pod starts and deleted when the Pod is removed. It is not stored anywhere permanent; it lives on the node's local disk (or in memory if you configure it that way). Its lifetime is tied to the Pod, not to any individual container inside it.

This makes emptyDir useful for two things: sharing data between containers in the same Pod, and preserving files across container restarts within that Pod.

Open a new file in nano:

```console
nano emptydir-pod.yaml
```

Type or paste the following content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "echo hello from writer > /shared/data.txt; sleep 3600"]
      volumeMounts:
        - name: shared
          mountPath: /shared
    - name: reader
      image: busybox
      command: ["sh", "-c", "sleep 5; cat /shared/data.txt; sleep 3600"]
      volumeMounts:
        - name: shared
          mountPath: /shared
  volumes:
    - name: shared
      emptyDir: {}
```

The `writer` container writes a file to `/shared/data.txt`. The `reader` waits 5 seconds (so the writer finishes first) then reads it. Both containers mount the same volume at `/shared`.

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the Pod:

```console
kubectl apply -f emptydir-pod.yaml
```

Wait for both containers to start:

```console
kubectl get pod emptydir-demo
```

Wait until `READY` shows `2/2`. Then check the reader's logs:

```console
kubectl logs emptydir-demo -c reader
```

Example output:

```console
hello from writer
```

The reader saw a file it never created. The emptyDir volume is the shared pipe between the two containers.

Now delete the Pod. The volume and its data disappear with it:

```console
kubectl delete pod emptydir-demo
```

There is no way to recover that data. For anything that needs to survive Pod deletion, you need a PersistentVolumeClaim.

---

## Step 2: PersistentVolumeClaim

A PersistentVolumeClaim (PVC) is a request for storage. You describe what you need (size, access mode) and a StorageClass provisions an actual volume to fulfill it. The volume lives independently of any Pod, so it survives Pod restarts, deletions, and rescheduling.

The access mode `ReadWriteOnce` means the volume can only be mounted read-write by a single node at a time.

Open a new file in nano:

```console
nano pvc.yaml
```

Type or paste the following content:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the PVC:

```console
kubectl apply -f pvc.yaml
```

Check its status:

```console
kubectl get pvc
```

Example output:

```console
NAME       STATUS    VOLUME   CAPACITY   ACCESS MODES
demo-pvc   Pending
```

`Pending` is expected here. The `local-path` StorageClass in this cluster uses `WaitForFirstConsumer` binding mode: it waits for a Pod to claim the PVC before provisioning the volume. This allows the scheduler to pick a node for the Pod first, then provision the volume on that same node. Without this, you could end up with a volume on node A and a Pod scheduled on node B, which would fail.

Describe the PVC to see the wait message:

```console
kubectl describe pvc demo-pvc
```

Look for this line in the Events section:

```console
waiting for first consumer to be created before binding
```

Create a Pod manifest that mounts the PVC:

```console
nano pvc-pod.yaml
```

Type or paste the following content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-demo
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: demo-pvc
```

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the Pod:

```console
kubectl apply -f pvc-pod.yaml
```

Wait for the Pod to be Running:

```console
kubectl get pod pvc-demo
```

Once the Pod is running, the StorageClass has provisioned a volume and bound it. Confirm the PVC status changed:

```console
kubectl get pvc demo-pvc
```

Example output:

```console
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES
demo-pvc   Bound    pvc-...                                    1Gi        RWO
```

`Bound` means the PVC is backed by a real volume. Now write a file to the volume manually:

```console
kubectl exec pvc-demo -- sh -c 'echo persistent > /data/file.txt'
```

Read it back to confirm it was written:

```console
kubectl exec pvc-demo -- cat /data/file.txt
```

Example output:

```console
persistent
```

Delete the Pod. This is the key test: will the data survive?

```console
kubectl delete pod pvc-demo
```

Recreate the Pod from the same manifest:

```console
kubectl apply -f pvc-pod.yaml
```

Wait for it to be Running:

```console
kubectl get pod pvc-demo
```

Read the file again:

```console
kubectl exec pvc-demo -- cat /data/file.txt
```

Example output:

```console
persistent
```

The file is still there. The container itself never wrote it during this second startup; it survived on the volume. This is the fundamental difference between a PVC and an emptyDir.

---

## Step 3: StatefulSet with volumeClaimTemplate

A StatefulSet is like a Deployment with two extra guarantees: each Pod gets a stable, predictable hostname (`web-0`, `web-1`, `web-2`) and each Pod gets its own dedicated PVC that follows it if it is rescheduled. This is what databases and message queues need.

A regular Deployment would give all Pods the same PVC. A StatefulSet's `volumeClaimTemplates` field creates a separate PVC for each Pod.

Open a new file in nano:

```console
nano statefulset.yaml
```

Type or paste the following content:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web"
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx
          volumeMounts:
            - name: data
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 100Mi
```

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the StatefulSet:

```console
kubectl apply -f statefulset.yaml
```

Watch the Pods start. Unlike a Deployment, a StatefulSet starts Pods in strict order: `web-0` must be Running and Ready before `web-1` starts, and so on:

```console
kubectl get pods -w
```

Press Ctrl+C once all three Pods are Running.

Check the PVCs. You should see one per Pod, named after the `volumeClaimTemplate` and the Pod:

```console
kubectl get pvc
```

Example output:

```console
NAME          STATUS   VOLUME    CAPACITY
data-web-0    Bound    ...       100Mi
data-web-1    Bound    ...       100Mi
data-web-2    Bound    ...       100Mi
```

If `web-1` is deleted and rescheduled, it will reattach to `data-web-1`, not to some other Pod's volume. Each Pod always gets its own persistent identity.

---

## Cleanup

Delete the manual Pod from Step 2:

```console
kubectl delete pod pvc-demo
```

Delete the PVC from Step 2:

```console
kubectl delete pvc demo-pvc
```

Delete the StatefulSet:

```console
kubectl delete statefulset web
```

Delete the StatefulSet's PVCs (StatefulSet deletion does not automatically delete them, which is intentional to protect data):

```console
kubectl delete pvc data-web-0 data-web-1 data-web-2
```

---
