# Lab 6: Pods and Labels

A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that share a network namespace and storage. In most production systems you never create Pods directly (a Deployment manages them for you), but understanding the raw Pod first makes everything else easier.

This lab covers creating a Pod, inspecting it, accessing it from the inside, and using labels and annotations to organise cluster objects.
---

## Step 0: Verify cluster access

Before touching anything, confirm kubectl is pointed at the right cluster:

```console
kubectl cluster-info
```

Example output:

```console
Kubernetes control plane is running at https://0.0.0.0:6443
CoreDNS is running at https://0.0.0.0:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

Check the client and server versions:

```console
kubectl version
```

Example output:

```console
Client Version: v1.36.2
Kustomize Version: v5.8.1
Server Version: v1.35.5+k3s1
```

List the nodes to understand the cluster topology:

```console
kubectl get nodes -o wide
```

Example output:

```console
NAME                        STATUS   ROLES           AGE   VERSION        INTERNAL-IP   EXTERNAL-IP   OS-IMAGE           KERNEL-VERSION    CONTAINER-RUNTIME
k3d-k8s-workshop-agent-0    Ready    <none>          41m   v1.35.5+k3s1   172.18.0.4    <none>        K3s v1.35.5+k3s1   6.17.0-1017-aws   containerd://2.2.3-k3s1
k3d-k8s-workshop-agent-1    Ready    <none>          41m   v1.35.5+k3s1   172.18.0.5    <none>        K3s v1.35.5+k3s1   6.17.0-1017-aws   containerd://2.2.3-k3s1
k3d-k8s-workshop-agent-2    Ready    <none>          41m   v1.35.5+k3s1   172.18.0.3    <none>        K3s v1.35.5+k3s1   6.17.0-1017-aws   containerd://2.2.3-k3s1
k3d-k8s-workshop-server-0   Ready    control-plane   41m   v1.35.5+k3s1   172.18.0.2    <none>        K3s v1.35.5+k3s1   6.17.0-1017-aws   containerd://2.2.3-k3s1
```

You have three worker nodes (`agent`) and one control-plane node (`server`). Most Pods will be scheduled on the agents.

List the existing namespaces:

```console
kubectl get namespaces
```

Example output:

```console
NAME              STATUS   AGE
default           Active   41m
ingress-nginx     Active   41m
kube-node-lease   Active   41m
kube-public       Active   41m
kube-system       Active   41m
```

Namespaces are logical partitions inside a cluster. Your work lives in `default` unless you specify otherwise.

---

## Step 1: Run a Pod

`kubectl run` is the imperative shortcut for creating a standalone Pod. It is the Kubernetes equivalent of `docker run`. The Pod is not managed by any higher-level controller, which is intentional for this exercise.

```console
kubectl run nginx --image=nginx
```

Example output:

```console
pod/nginx created
```

Wait a few seconds for the container image to be pulled and the container to start, then check the Pod status:

```console
kubectl get pods
```

Example output:

```console
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          10s
```

`1/1` means one container is running out of one total. `RESTARTS` shows how many times the container has crashed and been restarted. Zero is good.

---

## Step 2: Inspect the Pod

View the Pod's stdout logs. nginx writes its access log to stdout, so you will see startup messages:

```console
kubectl logs nginx
```

Now get a detailed view of the Pod including its configuration, status, and recent events:

```console
kubectl describe pod nginx
```

This output is long. The most useful part when diagnosing problems is the **Events** section at the very bottom. It shows what the scheduler and kubelet did to bring the Pod up: pulling the image, creating the container, and starting it. When a Pod is stuck, this is your first stop.

---

## Step 3: Exec into the Pod

You can open an interactive shell directly inside a running container. This is the equivalent of `docker exec`:

```console
kubectl exec -it nginx -- /bin/bash
```

You are now inside the container. You share its network namespace, meaning `localhost` refers to the container's own network interface.

Print the hostname of the container. Kubernetes sets the hostname to the Pod name by default:

```console
hostname
```

Example output:

```console
nginx
```

Send an HTTP request to the nginx server running in the same container. Because you are inside the container's network namespace, `localhost` hits nginx directly without going through a Kubernetes Service:

```console
curl http://localhost
```

Example output:

```console
<!DOCTYPE html>
...
```

When you're done exploring, exit the shell:

```console
exit
```

---

## Step 4: Add labels to the Pod

Labels are key-value pairs attached to Kubernetes objects. They are not just cosmetic. Services use label selectors to decide which Pods to route traffic to. Deployments use them to identify the Pods they manage. Being precise with labels is what keeps controllers from accidentally picking up the wrong Pods.

Add an `env=prod` label to the running Pod:

```console
kubectl label pod nginx env=prod
```

Example output:

```console
pod/nginx labeled
```

List all Pods and show their labels:

```console
kubectl get pods --show-labels
```

Example output:

```console
NAME    READY   STATUS    RESTARTS   AGE   LABELS
nginx   1/1     Running   0          2m    env=prod,run=nginx
```

Notice `run=nginx` was already there. kubectl run adds it automatically. Now filter Pods using a label selector:

```console
kubectl get pods -l env=prod
```

Only Pods carrying `env=prod` are returned. This is the same mechanism a Service uses to find its backends.

Add a second label:

```console
kubectl label pod nginx tier=frontend
```

Verify both labels are visible:

```console
kubectl get pods --show-labels
```

To change an existing label value, you must add `--overwrite`:

```console
kubectl label pod nginx env=staging --overwrite
```

Check that the value changed:

```console
kubectl get pods --show-labels
```

The `env` label now reads `staging`.

---

## Step 5: Add an annotation

Annotations also attach metadata to objects, but they are not used for selection. Think of them as a place for documentation, tool configuration, or audit data. A Service will never select Pods by annotation.

Add an annotation describing this Pod:

```console
kubectl annotate pod nginx description="workshop test pod"
```

Annotations live in the `metadata` section of the object. Pull just that section from the full YAML to confirm:

```console
kubectl get pod nginx -o yaml | grep -A4 annotations
```

You will see the annotation nested under `metadata.annotations`.

Example output:

```console
  annotations:
    cni.projectcalico.org/containerID: 25bb528e370c6cd38e69ee036d3ac0648d5d7a33a0fb37a05bf836df7588b751
    cni.projectcalico.org/podIP: 10.42.2.11/32
    cni.projectcalico.org/podIPs: 10.42.2.11/32
    description: workshop test pod # This is your annotation
```

---

## Step 6: Pull from Harbor using an image pull secret

The workshop Harbor registry is private. When Kubernetes tries to pull an image from a private registry it needs credentials. These credentials are stored as a Kubernetes Secret and referenced from the Pod spec. Without the secret, the pull fails.

The registry is already seeded with common images (nginx, busybox) so this step works without pushing anything first.

Create a registry credential secret:

```console
kubectl create secret docker-registry harbor-creds \
    --docker-server=harbor.ts-k8s-workshop.com \
    --docker-username=$USER \
    --docker-password=Kubernetes1!
```

Confirm the secret was created:

```console
kubectl get secret harbor-creds
```

Inspect the secret type. Kubernetes knows this is a registry credential from the `kubernetes.io/dockerconfigjson` type:

```console
kubectl describe secret harbor-creds
```

Now run a Pod that pulls the pre-seeded nginx image from Harbor. The `--overrides` flag injects extra fields into the generated Pod spec:

```console
kubectl run harbor-test \
    --image=harbor.ts-k8s-workshop.com/workshop/nginx:latest \
    --overrides='{"spec":{"imagePullSecrets":[{"name":"harbor-creds"}]}}'
```

Watch the Pod start. If the credentials are correct the status moves through `ContainerCreating` to `Running`:

```console
kubectl get pod harbor-test
```

If something goes wrong, describe the Pod to see the pull error:

```console
kubectl describe pod harbor-test
```

Without the pull secret, the pull fails with `ErrImagePull`. In production YAML, the secret reference looks like this:

```yaml
spec:
  imagePullSecrets:
    - name: harbor-creds
  containers:
    - name: app
      image: harbor.ts-k8s-workshop.com/workshop/nginx:latest
```

---

## Step 7: Clean up

Delete both Pods by name:

```console
kubectl delete pod nginx harbor-test
```

Example output:

```console
pod "nginx" deleted
pod "harbor-test" deleted
```

Confirm no Pods remain:

```console
kubectl get pods
```

Example output:

```console
No resources found in default namespace.
```

---
