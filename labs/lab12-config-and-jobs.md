# Lab 12: Secrets, ConfigMaps, and Jobs

Applications need configuration (database URLs, feature flags) and credentials (passwords, API keys). Baking these into container images is a bad practice: it forces a rebuild for every config change and exposes secrets to anyone with registry access. Kubernetes separates configuration from the image using ConfigMaps and Secrets.

This lab also covers Jobs (one-shot batch work), CronJobs (scheduled batch work), init containers, and the native sidecar pattern.
---

## Part A: Secrets

### Step 1: Create a Secret

A Secret stores sensitive data as base64-encoded key-value pairs. Base64 is encoding, not encryption. A Secret's value is trivially decoded with `base64 -d`. Kubernetes stores Secrets in etcd, and encryption at rest requires additional cluster configuration. The practical security benefit of Secrets over ConfigMaps is access control: you can grant a workload access to Secrets without granting it access to ConfigMaps, and vice versa.

Create a Secret with two literal values:

```console
kubectl create secret generic my-secret \
    --from-literal=username=admin \
    --from-literal=password=supersecret
```

Check the Secret exists:

```console
kubectl get secret my-secret
```

Describe it. Notice the values appear as `<hidden>` in the describe output:

```console
kubectl describe secret my-secret
```

Get the raw YAML to see the base64-encoded values directly:

```console
kubectl get secret my-secret -o yaml
```

The `data` section holds the base64-encoded values. You can decode them with `echo <value> | base64 -d`. This is a reminder that base64 is not security; proper RBAC is.

### Step 2: Mount Secret as environment variables

The most common way to inject Secrets into an application is as environment variables. The values are injected at container start time from the Secret object.

Open a new file in nano:

```console
nano secret-pod.yaml
```

Type or paste the following content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "echo username=$USERNAME; echo password=$PASSWORD; sleep 3600"]
      env:
        - name: USERNAME
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: username
        - name: PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: password
```

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the Pod:

```console
kubectl apply -f secret-pod.yaml
```

Wait for it to be Running:

```console
kubectl get pod secret-env
```

Check the logs to confirm both env vars were injected:

```console
kubectl logs secret-env
```

Example output:

```console
username=admin
password=supersecret
```

One limitation of env-var injection: if the Secret changes, running containers do not see the new values. They need to be restarted to pick up the change. Volume mounts (next step) update dynamically.

### Step 3: Mount Secret as a volume

Mounting a Secret as a volume creates one file per key at the specified mount path. The values are written as plain text (decoded from base64). Unlike environment variables, Kubernetes updates mounted Secret files automatically when the Secret changes, without restarting the container.

Open a new file in nano:

```console
nano secret-volume-pod.yaml
```

Type or paste the following content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-vol
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "ls /var/run/secrets/my-secret/; cat /var/run/secrets/my-secret/username; echo; sleep 3600"]
      volumeMounts:
        - name: secret-vol
          mountPath: /var/run/secrets/my-secret
          readOnly: true
  volumes:
    - name: secret-vol
      secret:
        secretName: my-secret
```

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the Pod:

```console
kubectl apply -f secret-volume-pod.yaml
```

Wait for it to be Running:

```console
kubectl get pod secret-vol
```

Check the logs:

```console
kubectl logs secret-vol
```

Example output:

```console
password
username
admin
```

The first two lines are from `ls`: each Secret key became a file at the mount path. The third line is the content of the `username` file: the actual secret value, decoded from base64 and written as plain text.

---

## Part B: ConfigMaps

### Step 4: Create a ConfigMap

A ConfigMap stores non-sensitive configuration data. The API is identical to Secrets, but the values are stored in plain text and not restricted by RBAC in the same way. Use ConfigMaps for anything that is safe to read: app settings, feature flags, environment identifiers.

Create a ConfigMap with two values:

```console
kubectl create configmap app-config \
    --from-literal=SPECIAL_LEVEL=very \
    --from-literal=SPECIAL_TYPE=charm
```

Confirm it was created:

```console
kubectl get configmap app-config
```

### Step 5: Mount ConfigMap as a volume

Mounting a ConfigMap as a volume works the same way as mounting a Secret: each key becomes a file at the mount path.

Open a new file in nano:

```console
nano configmap-pod.yaml
```

Type or paste the following content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-vol
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "ls /etc/config/; cat /etc/config/SPECIAL_LEVEL; echo; sleep 3600"]
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: app-config
```

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the Pod:

```console
kubectl apply -f configmap-pod.yaml
```

Wait for it to be Running:

```console
kubectl get pod configmap-vol
```

Check the output:

```console
kubectl logs configmap-vol
```

Example output:

```console
SPECIAL_LEVEL
SPECIAL_TYPE
very
```

The pattern is the same as the Secret volume: keys become filenames, values become file content.

---

## Part C: Jobs

### Step 6: Create a Job

A Job runs a Pod to completion. Unlike a Deployment (which keeps Pods running indefinitely), a Job is done when its container exits successfully. Use Jobs for database migrations, batch processing, report generation, or any one-shot task.

The `restartPolicy: Never` means if the container fails, a new Pod is created for the retry (up to `backoffLimit`). Setting it to `OnFailure` would restart the container inside the same Pod instead.

Open a new file in nano:

```console
nano job.yaml
```

Type or paste the following content:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: hello
          image: busybox
          command: ["echo", "hello from job"]
```

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the Job:

```console
kubectl apply -f job.yaml
```

Check the Job status. The `COMPLETIONS` column shows `0/1` while running and `1/1` when done:

```console
kubectl get jobs
```

Check the Pod the Job created:

```console
kubectl get pods
```

The Pod status will be `Completed`. Kubernetes keeps completed Job Pods around so you can read the logs.

Read the Job output:

```console
kubectl logs -l job-name=hello-job
```

Example output:

```console
hello from job
```

### Step 7: Create a CronJob

A CronJob wraps a Job with a cron schedule. Kubernetes creates a new Job (and a new Pod) on each firing. The schedule uses standard five-field cron syntax.

Open a new file in nano:

```console
nano cronjob.yaml
```

Type or paste the following content:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: date-cron
spec:
  schedule: "*/1 * * * *"
  successfulJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: date
              image: busybox
              command: ["sh", "-c", "date"]
```

`*/1 * * * *` means "every minute". `successfulJobsHistoryLimit: 3` keeps only the last 3 completed Job objects (and their Pods) to avoid clutter.

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the CronJob:

```console
kubectl apply -f cronjob.yaml
```

Check it was created:

```console
kubectl get cronjobs
```

Watch for a Job to appear. The first one fires within the next minute:

```console
kubectl get jobs -w
```

Once a Job appears, press Ctrl+C. Get its name from the list, then check its logs:

```console
kubectl logs -l job-name=<job-name>
```

The output will be the current date and time from inside the container.

---

## Part D: Init Containers and Sidecars

### Step 8: Init container that gates startup

An init container runs to completion before any main container starts. If the init container fails, Kubernetes retries it. The main container never starts until all init containers have succeeded. Use init containers to wait for a dependency to become ready, or to populate a shared volume with data the app needs at startup.

Open a new file in nano:

```console
nano init-pod.yaml
```

Type or paste the following content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
    - name: init-writer
      image: busybox
      command: ["sh", "-c", "echo 'ready' > /shared/status.txt; echo init done"]
      volumeMounts:
        - name: shared
          mountPath: /shared
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "echo app started; cat /shared/status.txt; sleep 3600"]
      volumeMounts:
        - name: shared
          mountPath: /shared
  volumes:
    - name: shared
      emptyDir: {}
```

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the Pod:

```console
kubectl apply -f init-pod.yaml
```

Watch the Pod status. It moves through `Init:0/1` (init container running) before transitioning to `Running` (main container):

```console
kubectl get pod init-demo -w
```

Press Ctrl+C once the Pod is Running.

Check the main container's output. It read the file that the init container wrote:

```console
kubectl logs init-demo -c app
```

Example output:

```console
app started
ready
```

The init container wrote the file; the app container read it. If the init container had failed, the app would never have started.

---

### Step 9: Native sidecar container

Kubernetes 1.29+ supports native sidecar containers: init containers with `restartPolicy: Always`. Unlike regular init containers (which run once and exit), a native sidecar starts before the main container and keeps running alongside it for the entire lifetime of the Pod. This is the right pattern for log shippers, metrics collectors, and service mesh proxies that need to outlive any single main container restart.

Open a new file in nano:

```console
nano sidecar-pod.yaml
```

Type or paste the following content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:
  initContainers:
    - name: log-shipper
      image: busybox
      restartPolicy: Always
      command: ["sh", "-c", "while true; do echo '[log-shipper] heartbeat'; sleep 5; done"]
      volumeMounts:
        - name: logs
          mountPath: /logs
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "i=0; while true; do echo \"request $i\" >> /logs/app.log; i=$((i+1)); sleep 3; done"]
      volumeMounts:
        - name: logs
          mountPath: /logs
  volumes:
    - name: logs
      emptyDir: {}
```

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the Pod:

```console
kubectl apply -f sidecar-pod.yaml
```

Check that both containers are running:

```console
kubectl get pod sidecar-demo
```

Native sidecars are represented as init containers in the Kubernetes API (because of how the feature was added). Check their status directly:

```console
kubectl get pod sidecar-demo -o jsonpath='{.status.initContainerStatuses[*].name}{" "}{.status.containerStatuses[*].name}{"\n"}'
```

Example output:

```console
log-shipper app
```

`log-shipper` appears in `initContainerStatuses` because it is technically an init container with `restartPolicy: Always`.

Read the sidecar's own stdout:

```console
kubectl logs sidecar-demo -c log-shipper
```

Read the log file the app container wrote to the shared volume:

```console
kubectl exec sidecar-demo -c log-shipper -- cat /logs/app.log
```

The sidecar has full access to everything written to `/logs` by the main container. This is the pattern used by log forwarders (Fluentd, Fluent Bit) and service mesh proxies (Envoy) in production.

---

## Cleanup

Delete all Pods:

```console
kubectl delete pod secret-env secret-vol configmap-vol init-demo sidecar-demo
```

Delete the Secret and ConfigMap:

```console
kubectl delete secret my-secret
```

```console
kubectl delete configmap app-config
```

Delete the Job and CronJob:

```console
kubectl delete job hello-job
```

```console
kubectl delete cronjob date-cron
```

---
