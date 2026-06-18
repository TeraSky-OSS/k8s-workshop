# Lab 13: Troubleshooting

Production Kubernetes clusters fail in predictable ways. This lab deliberately breaks things and walks through the diagnostic toolkit for the most common failure modes. Each step maps to something you will encounter in a real environment.

The diagnostic order is almost always the same: check Pod status, read the Events, read the logs. Most issues reveal themselves within those three steps.
---

## Step 1: Diagnose ErrImagePull

The image pull failure is the most common early-stage problem. It happens whenever the image name is wrong, the tag does not exist, or the registry is unreachable.

Create a Pod with a nonexistent image tag:

```console
kubectl run broken-image --image=nginx:doesnotexist999
```

Check the status:

```console
kubectl get pod broken-image
```

Example output:

```console
NAME           READY   STATUS         RESTARTS   AGE
broken-image   0/1     ErrImagePull   0          4s
```

`ErrImagePull` means the first pull attempt just failed. If you wait, it will move to `ImagePullBackOff`, which means Kubernetes is pausing before retrying (the backoff grows from 10 seconds up to 5 minutes). Both statuses point to the same root cause.

Find the exact error by reading the Events:

```console
kubectl describe pod broken-image
```

Scroll to the Events section at the bottom. You will see something like:

```console
Failed to pull image "nginx:doesnotexist999": ... not found
```

That specific message tells you whether the tag does not exist, the image name is misspelled, or the registry returned an authentication error. Now fix it by deleting the broken Pod and creating a new one with the correct image:

```console
kubectl delete pod broken-image
```

```console
kubectl run fixed-image --image=nginx:latest
```

Confirm it starts cleanly:

```console
kubectl get pod fixed-image
```

---

## Step 2: Diagnose CrashLoopBackOff

A `CrashLoopBackOff` means the container is starting, failing, and being restarted repeatedly. Kubernetes applies exponential backoff (2s, 4s, 8s... up to 5 minutes between restarts) to avoid hammering a broken container. The container is not gone, it is just waiting to be retried.

Create a Pod that logs a message and then exits with an error code:

```console
kubectl run crasher --image=busybox -- /bin/sh -c "echo 'starting up'; sleep 1; echo 'fatal error'; exit 1"
```

Watch it restart. Wait until `RESTARTS` reaches at least 1, then press Ctrl+C:

```console
kubectl get pod crasher -w
```

Check the current container's logs first:

```console
kubectl logs crasher
```

Example output:

```console
starting up
fatal error
```

The `--previous` flag retrieves logs from the last dead container, not the currently running one. This is useful once the Pod has already restarted at least once. If you run it too early (before the first restart), it returns nothing. Run it now:

```console
kubectl logs crasher --previous
```

Example output (after at least one restart):

```console
starting up
fatal error
```

Now describe the Pod and look at the container state for the exit code:

```console
kubectl describe pod crasher
```

Find the section that looks like:

```console
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
```

The exit code is the first clue in a real crash. Exit code `1` is a generic application error. Code `137` is an OOMKill (the container was killed for exceeding its memory limit). Code `139` is a segfault.

Fix the Pod by deleting the crashing one and creating a working version:

```console
kubectl delete pod crasher
```

```console
kubectl run working --image=busybox -- /bin/sh -c "echo ok; sleep 3600"
```

Verify it stays running:

```console
kubectl get pod working
```

---

## Step 3: Connectivity test using busybox

When a Service or Deployment is misbehaving, you often need to test connectivity from inside the cluster, because ClusterIP addresses are only reachable within the cluster. A temporary busybox Pod is the standard tool for this.

This step demonstrates the exact diagnostic flow. There is intentionally no `web-svc` Service in your cluster. The commands are expected to fail, and the failure messages tell you where the problem is.

Run a temporary busybox Pod and attach a shell:

```console
kubectl run nettest --image=busybox --rm -it -- sh
```

Once inside the shell, test DNS resolution for the Service:

```console
nslookup web-svc
```

Example output (DNS failure, the Service does not exist):

```console
Server:    10.43.0.10
Address 1: 10.43.0.10 kube-dns.kube-system.svc.cluster.local

nslookup: can't resolve 'web-svc'
```

DNS failure means the Service itself does not exist (or the name is wrong).

Now test the connection:

```console
wget -qO- http://web-svc
```

Example output:

```console
wget: bad address 'web-svc'
```

In a real scenario: if `nslookup` passes but `wget` times out, the Service exists but has no healthy Pods behind it (check `kubectl get endpoints web-svc`). If both fail, the Service itself is missing or misconfigured.

Exit the temporary Pod:

```console
exit
```

---

## Step 4: Use kubectl get events

Cluster-level events capture a history of what happened across all resources. They are ephemeral (they expire after about an hour) and are your fastest way to see recent failures.

List events sorted by most recent last:

```console
kubectl get events --sort-by='.lastTimestamp'
```

Filter for warnings only. These are the events that signal something went wrong:

```console
kubectl get events --field-selector type=Warning
```

You should see the image pull error and the crash events from Steps 1 and 2.

---

## Step 5: Check resource usage

If a Pod is running but behaving badly (slow, being OOMKilled, throttled), the metrics-server gives you a quick view of actual CPU and memory consumption:

```console
kubectl top nodes
```

Check Pod-level usage:

```console
kubectl top pods
```

If a Pod's memory is approaching its limit, it will soon be OOMKilled. If its CPU is being heavily throttled, you will see high latency without CrashLoopBackOff. Both are diagnosed here before they become incidents.

---

## Step 6: Debug a Pod that won't start

Sometimes the container's entrypoint itself is crashing before you get a chance to exec into it. The fix is to override the entrypoint with a long-running no-op command, which gives you a shell to explore the filesystem, test the real command manually, and check for missing files or configuration.

Run nginx with its entrypoint replaced by `sleep`:

```console
kubectl run debug-nginx \
    --image=nginx \
    --command \
    -- sleep 3600
```

Exec into the running container:

```console
kubectl exec -it debug-nginx -- /bin/bash
```

From inside, you can run the real entrypoint manually, inspect the filesystem, check environment variables, or test connectivity. This is how you debug startup failures that crash too fast to read logs.

When finished, exit:

```console
exit
```

---

## Cleanup

Delete the Pods left running:

```console
kubectl delete pod fixed-image working debug-nginx
```

---
