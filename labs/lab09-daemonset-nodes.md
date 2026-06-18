# Lab 9: DaemonSets and Node Scheduling

By default, the scheduler places Pods wherever there is capacity. Sometimes you need more control: a logging agent on every node, a workload pinned to nodes with specific hardware, or a node you want to reserve for certain workloads only. This lab covers the building blocks for all of that.
---

## Step 1: Create a ReplicaSet

A ReplicaSet keeps a specified number of Pod replicas running at all times. You rarely create ReplicaSets directly in production (a Deployment manages them for you and adds rolling updates), but understanding them is useful before seeing how DaemonSets differ.

Create the file:

```console
nano rs.yaml
```

Paste the content below, then save with **Ctrl+O → Enter** and exit with **Ctrl+X**:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-rs
  template:
    metadata:
      labels:
        app: web-rs
    spec:
      containers:
        - name: nginx
          image: nginx:1.30
```

Apply the manifest:

```console
kubectl apply -f rs.yaml
```

Confirm the ReplicaSet created two Pods:

```console
kubectl describe replicaset web-rs
```

Check where the Pods landed. The `-o wide` flag adds a NODE column:

```console
kubectl get pods -l app=web-rs -o wide
```

The scheduler spread the two Pods across different nodes to improve availability. There is no control over which node each Pod goes to, and there is no guarantee of one Pod per node.

---

## Step 2: Convert to a DaemonSet

A DaemonSet guarantees exactly one Pod per eligible node. It is the right choice for node-level infrastructure: log collectors, monitoring agents, network proxies. You do not set a replica count; the count is determined by how many nodes exist.

Open the file for editing:

```console
nano rs.yaml
```

Make the following changes, then save with **Ctrl+O → Enter** and exit with **Ctrl+X**:

- Change `kind: ReplicaSet` to `kind: DaemonSet`
- Change `name: web-rs` to `name: web-ds`
- Remove the `replicas: 2` line entirely (DaemonSets do not have a replica count)
- Change both `app: web-rs` label values to `app: web-ds` to avoid conflicts with the old ReplicaSet

Delete the old ReplicaSet first:

```console
kubectl delete replicaset web-rs
```

Apply the DaemonSet:

```console
kubectl apply -f rs.yaml
```

List the Pods with their node assignments:

```console
kubectl get pods -l app=web-ds -o wide
```

You should see exactly three Pods, one on each agent node. The control-plane node is skipped because k3d taints it with `NoSchedule` by default, which the DaemonSet does not tolerate.

---

## Step 3: Label a node and use nodeSelector

Labels are not just for Pods. You can label nodes too, and then use `nodeSelector` in the Pod spec to restrict which nodes a Pod can land on. This is the simplest form of node affinity.

Add a `disktype=ssd` label to the first agent node:

```console
kubectl label node k3d-k8s-workshop-agent-0 disktype=ssd
```

Confirm the label was applied:

```console
kubectl get nodes --show-labels | grep disktype
```

Create the Pod manifest:

```console
nano pod-ssd.yaml
```

Paste the content below, then save with **Ctrl+O → Enter** and exit with **Ctrl+X**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  nodeSelector:
    disktype: ssd
  containers:
    - name: nginx
      image: nginx
```

Apply the Pod:

```console
kubectl apply -f pod-ssd.yaml
```

Check which node the Pod landed on:

```console
kubectl get pod ssd-pod -o wide
```

It should be on `k3d-k8s-workshop-agent-0`, the only node carrying the `disktype=ssd` label. If no node had the label, the Pod would stay `Pending` indefinitely.

---

## Step 4: Apply a taint and add a toleration

Taints repel Pods. A node with a `NoSchedule` taint refuses any Pod that does not explicitly tolerate it. Tolerations are the matching opt-in: a Pod that declares the right toleration is allowed on the tainted node.

The pattern is useful for reserving nodes for specific workloads (for example, GPU nodes that you only want ML jobs on, or nodes with licensed software).

Add a `NoSchedule` taint to the second agent node:

```console
kubectl taint nodes k3d-k8s-workshop-agent-1 workshop=reserved:NoSchedule
```

Create a Pod manifest that targets the tainted node but has **no** toleration:

```console
nano pod-no-toleration.yaml
```

Paste the content below, then save with **Ctrl+O → Enter** and exit with **Ctrl+X**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-toleration
spec:
  nodeSelector:
    kubernetes.io/hostname: k3d-k8s-workshop-agent-1
  containers:
    - name: busybox
      image: busybox
      command: ["sleep", "3600"]
```

Apply it:

```console
kubectl apply -f pod-no-toleration.yaml
```

Check its status. The `nodeSelector` forces the scheduler to try the tainted node, but the missing toleration means it cannot be placed there:

```console
kubectl get pod no-toleration
```

Example output:

```console
NAME             READY   STATUS    RESTARTS   AGE
no-toleration    0/1     Pending   0          5s
```

Look at the Events to see the exact rejection message:

```console
kubectl describe pod no-toleration | grep -A3 Events
```

You will see: `1 node(s) had untolerated taint {workshop: reserved}`.

Now create a Pod with the matching toleration:

```console
nano pod-toleration.yaml
```

Paste the content below, then save with **Ctrl+O → Enter** and exit with **Ctrl+X**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  nodeSelector:
    kubernetes.io/hostname: k3d-k8s-workshop-agent-1
  tolerations:
    - key: "workshop"
      operator: "Equal"
      value: "reserved"
      effect: "NoSchedule"
  containers:
    - name: nginx
      image: nginx
```

Apply it:

```console
kubectl apply -f pod-toleration.yaml
```

Confirm it scheduled on the tainted node:

```console
kubectl get pod with-toleration -o wide
```

The toleration key, value, and effect must all match the taint exactly. With a match, the scheduler treats the tainted node like any other eligible node.

---

## Step 5: Clean up

Remove the taint from the node:

```console
kubectl taint nodes k3d-k8s-workshop-agent-1 workshop:NoSchedule-
```

The trailing `-` on the taint key removes it.

Delete the Pods:

```console
kubectl delete pod no-toleration with-toleration ssd-pod
```

Delete the DaemonSet:

```console
kubectl delete daemonset web-ds
```

Remove the node label:

```console
kubectl label node k3d-k8s-workshop-agent-0 disktype-
```

---
