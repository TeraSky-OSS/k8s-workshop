# Lab 8: Pod Configuration and Probes

Running a container without telling Kubernetes how many resources it needs, or whether it is actually healthy, is a recipe for noisy neighbour problems and silent failures. This lab adds resource requests and limits to a Deployment, then configures liveness and readiness probes so Kubernetes can manage the container lifecycle correctly.
---

## Step 1: Add resource requests and limits

**Requests** tell the scheduler how much CPU and memory to reserve on a node for this container. The scheduler will never place a Pod on a node that can't satisfy its requests. **Limits** are the enforcement ceiling: the container runtime kills or throttles the container if it exceeds them.

CPU is measured in millicores (m). `100m` is one tenth of a CPU core; `1000m` equals one full core. Memory is in bytes; `128Mi` is 128 mebibytes.

Open a new file in nano:

```console
nano nginx-resources.yaml
```

Type or paste the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-resources
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-resources
  template:
    metadata:
      labels:
        app: nginx-resources
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
```

Save the file: press Ctrl+O, then Enter. Exit nano: press Ctrl+X.

Apply the Deployment:

```console
kubectl apply -f nginx-resources.yaml
```

Confirm the Pod was created:

```console
kubectl get pods -l app=nginx-resources
```

Verify the requests and limits appear in the Pod spec:

```console
kubectl describe pod -l app=nginx-resources | grep -A6 Limits
```

You should see both the `Limits` and `Requests` sections with the values you set.

---

## Step 2: Add liveness and readiness probes

**Liveness probes** answer the question "is this container still functioning?" If the probe fails, Kubernetes restarts the container. Use it to recover from deadlocks or corrupt state.

**Readiness probes** answer "is this container ready to accept traffic?" If the probe fails, the container's IP is removed from the Service's endpoint list. Traffic stops reaching it, but the container is not restarted. Use it to gate traffic until the app has finished warming up, or to temporarily take a Pod out of rotation.

The two probes are independent. A container can be alive (liveness passes) but not ready (readiness fails), which is the expected state during a slow startup.

Open the file in nano:

```console
nano nginx-resources.yaml
```

Use Ctrl+W to search for `limits:`. Position the cursor after the last line under `limits` (after `memory: "256Mi"`). Press Enter and add the following lines at the same indentation level as `resources:` (10 spaces):

```yaml
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
```

`initialDelaySeconds` gives the container time to start before the first check. `periodSeconds` controls how often the probe runs.

Save the file: press Ctrl+O, then Enter. Exit nano: press Ctrl+X.

Apply the updated manifest:

```console
kubectl apply -f nginx-resources.yaml
```

Verify both probes appear in the Pod description:

```console
kubectl describe pod -l app=nginx-resources | grep -A8 "Liveness\|Readiness"
```

---

## Step 3: Observe probe behaviour

Watch the Pod while it starts. The readiness probe gates the Pod's entry into the Service's endpoint list, so the Pod briefly shows `0/1 READY` before the readiness check passes:

```console
kubectl get pods -w
```

Once the Pod shows `1/1 Running`, get its name for the next command:

```console
kubectl get pods -l app=nginx-resources
```

Now deliberately break the nginx server inside the Pod. Replace `<pod-name>` with the name from above:

```console
kubectl exec -it <pod-name> -- nginx -s stop
```

This sends a SIGQUIT to the nginx master process, stopping it. The liveness probe will now fail because GET / on port 80 returns a connection refused.

Watch what happens. The liveness probe will fail after a few checks and Kubernetes will restart the container:

```console
kubectl get pods -w
```

You should see the `RESTARTS` counter increment from 0 to 1. The container starts fresh from the image, with nginx running again. Press Ctrl+C when done.

---

## Step 4: Explore imagePullPolicy

`imagePullPolicy` controls whether Kubernetes pulls the image from the registry on every Pod start or reuses a locally cached copy.

- `Always`: always contact the registry, even if the image is cached. Safe for tags like `latest` that change over time.
- `IfNotPresent`: skip the registry pull if the image is already on the node. The default for tags that are not `latest`.
- `Never`: never pull; fail if the image is not already present on the node.

Open the file in nano:

```console
nano nginx-resources.yaml
```

Use Ctrl+W to search for `image: nginx`. Add a new line directly below it, at the same indentation level:

```yaml
          imagePullPolicy: IfNotPresent
```

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the change:

```console
kubectl apply -f nginx-resources.yaml
```

Confirm the pull policy appears in the container spec:

```console
kubectl get pod -l app=nginx-resources -o yaml | grep "Image\|Pull"
```

---

## Step 5: Create a Horizontal Pod Autoscaler

An HPA watches a Deployment's average CPU utilisation and adjusts the replica count automatically. When CPU rises above the target, it scales up; when it drops, it scales back down. It relies on the metrics-server to get real-time usage data from kubelets.

Check that metrics-server is working before proceeding:

```console
kubectl top nodes
```

If the command returns data, metrics-server is available. If it returns an error, skip this step and ask the instructor.

The HPA needs a Service to route load generator traffic to the Pods. Create one:

```console
kubectl expose deployment nginx-resources --port=80 --name=nginx-resources-svc
```

Create an HPA targeting 50% average CPU utilisation across all Pods in the Deployment:

```console
kubectl autoscale deployment nginx-resources --cpu=50% --min=1 --max=5
```

View the HPA. The `TARGETS` column shows current usage vs the target. It may read `<unknown>` for a minute while metrics-server collects initial data:

```console
kubectl get hpa nginx-resources
```

Example output (once metrics are available):

```console
NAME              REFERENCE                    TARGETS   MINPODS   MAXPODS   REPLICAS
nginx-resources   Deployment/nginx-resources   0%/50%    1         5         1
```

CPU is at 0%, so the HPA holds the Deployment at 1 replica. Now generate load. This Pod runs a tight wget loop against the Service in the background:

```console
kubectl run load-generator --image=busybox --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://nginx-resources-svc; done"
```

Watch the HPA. The metrics-server scrapes every 15 seconds and the HPA evaluates every 15 seconds, so it can take up to a minute before the `TARGETS` column rises and the replica count increases. Keep watching:

```console
kubectl get hpa nginx-resources -w
```

Example output as scaling happens:

```console
NAME              REFERENCE                    TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
nginx-resources   Deployment/nginx-resources   cpu: 0%/50%   1         5         1          24s
nginx-resources   Deployment/nginx-resources   cpu: 1%/50%   1         5         1          31s
nginx-resources   Deployment/nginx-resources   cpu: 25%/50%   1         5         1          46s
nginx-resources   Deployment/nginx-resources   cpu: 150%/50%   1         5         1          61s
nginx-resources   Deployment/nginx-resources   cpu: 148%/50%   1         5         3          76s
```

Press Ctrl+C when you see the replica count increase. Confirm the new Pods are running:

```console
kubectl get pods -l app=nginx-resources
```

Now stop the load generator:

```console
kubectl delete pod load-generator
```

Watch the HPA again. The HPA has a built-in 5-minute cooldown before scaling back down to prevent flapping, so it will take a few minutes for the replica count to return to 1:

```console
kubectl get hpa nginx-resources -w
```

Press Ctrl+C once you've seen it hold steady. The scale-down will complete on its own.

---

## Cleanup

Delete the Deployment, the Service, and the HPA:

```console
kubectl delete deployment nginx-resources
```

```console
kubectl delete service nginx-resources-svc
```

```console
kubectl delete hpa nginx-resources
```

---
