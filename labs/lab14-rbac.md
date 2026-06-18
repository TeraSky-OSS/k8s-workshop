# Lab 14: RBAC

By default, a Pod can do almost nothing against the Kubernetes API. RBAC (Role-Based Access Control) is how you grant it permissions. The model has three parts: a **ServiceAccount** (the identity), a **Role** (the permissions), and a **RoleBinding** (the link between them).

A ServiceAccount is not a human user account. It is an identity for a workload. Kubernetes automatically mounts a ServiceAccount token as a file inside every Pod. When a Pod calls the Kubernetes API, it authenticates with that token.
---

## Step 1: Run kubectl from inside a Pod with no permissions

The `default` ServiceAccount exists in every namespace and has no permissions. This is intentional: least privilege by default. You have to explicitly grant permissions to anything that needs them.

Create a Pod that runs kubectl inside the cluster, using the default ServiceAccount:

```console
kubectl run kubectl-pod \
    --image=cgr.dev/chainguard/kubectl:latest-dev \
    --command -- sleep 3600
```

Wait for it to be Running:

```console
kubectl get pod kubectl-pod
```

Exec into it and try to list Pods. The `kubectl` binary inside the container will use the mounted ServiceAccount token to authenticate:

```console
kubectl exec -it kubectl-pod -- kubectl get pods
```

Example output:

```console
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:<your-namespace>:default" cannot list resource "pods"
```

The error message is precise: it names the exact ServiceAccount (`system:serviceaccount:default:default`) and the exact action it was denied (`list resource "pods"`). This is the format you will see in real RBAC errors.

---

## Step 2: Create a read-only ServiceAccount

You will now build the RBAC chain from scratch: ServiceAccount, Role, and RoleBinding.

Create the ServiceAccount:

```console
kubectl create serviceaccount read-only
```

A Role defines a set of allowed actions (verbs) on a set of resources. Roles are namespace-scoped: they only apply within the namespace where they are created.

Open a new file in nano for the Role manifest:

```console
nano role-read-pods.yaml
```

Type or paste the following content:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
```

The empty string `""` for `apiGroups` refers to the core Kubernetes API group (Pods, Services, ConfigMaps, Secrets, etc.). Non-core resources use their group name, for example `apps` for Deployments.

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the Role:

```console
kubectl apply -f role-read-pods.yaml
```

Confirm the Role was created:

```console
kubectl get role pod-reader
```

A RoleBinding attaches a Role to a subject. The subject here is the `read-only` ServiceAccount in the `default` namespace:

```console
kubectl create rolebinding read-pods-binding \
    --role=pod-reader \
    --serviceaccount=default:read-only
```

The `--serviceaccount` format is `<namespace>:<serviceaccount-name>`. Getting this wrong is the most common RBAC mistake.

---

## Step 3: Verify read-only access

Create a Pod using the `read-only` ServiceAccount:

```console
kubectl run reader-pod \
    --image=cgr.dev/chainguard/kubectl:latest-dev \
    --overrides='{"spec":{"serviceAccountName":"read-only"}}' \
    --command -- sleep 3600
```

Wait for it to be Running:

```console
kubectl get pod reader-pod
```

Exec in and test listing Pods:

```console
kubectl exec -it reader-pod -- kubectl get pods
```

Example output:

```console
NAME          READY   STATUS
reader-pod    1/1     Running
kubectl-pod   1/1     Running
```

It works. Now test deleting a Pod (which the `pod-reader` Role does not allow):

```console
kubectl exec -it reader-pod -- kubectl delete pod kubectl-pod
```

Example output:

```console
Error from server (Forbidden): pods "kubectl-pod" is forbidden: User ... cannot delete resource "pods"
```

List is allowed; delete is denied. The Role grants only `get`, `watch`, and `list`.

---

## Step 4: Create a read-write ServiceAccount

Create a second ServiceAccount with broader permissions:

```console
kubectl create serviceaccount read-write
```

Open a new file in nano for the Role manifest:

```console
nano role-manager.yaml
```

Type or paste the following content:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-manager
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list", "create", "delete"]
```

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the Role:

```console
kubectl apply -f role-manager.yaml
```

Bind it to the `read-write` ServiceAccount:

```console
kubectl create rolebinding manage-pods-binding \
    --role=pod-manager \
    --serviceaccount=default:read-write
```

Create a Pod using the `read-write` ServiceAccount:

```console
kubectl run writer-pod \
    --image=cgr.dev/chainguard/kubectl:latest-dev \
    --overrides='{"spec":{"serviceAccountName":"read-write"}}' \
    --command -- sleep 3600
```

Wait for it to be Running:

```console
kubectl get pod writer-pod
```

Test that it can delete a Pod. `kubectl-pod` from Step 1 is still running:

```console
kubectl exec -it writer-pod -- kubectl delete pod kubectl-pod
```

Example output:

```console
pod "kubectl-pod" deleted from default namespace
```

The `pod-manager` Role allows `delete`, so this succeeds.

---

## Step 5: Use kubectl auth can-i

`kubectl auth can-i` lets you check permissions without actually attempting an operation. It is the fastest way to diagnose RBAC issues without running a real command and waiting for a Forbidden error.

Check your own admin permissions first:

```console
kubectl auth can-i get pods
```

Example output: `yes`

```console
kubectl auth can-i delete pods
```

Example output: `yes`

```console
kubectl auth can-i create deployments
```

Example output: `yes`

Now check permissions for the `read-only` ServiceAccount using `--as` to impersonate it:

```console
kubectl auth can-i get pods \
    --as=system:serviceaccount:default:read-only
```

Example output: `yes`

```console
kubectl auth can-i delete pods \
    --as=system:serviceaccount:default:read-only
```

Example output: `no`

This is the exact check to run when a workload is getting Forbidden errors. You do not need to recreate the scenario; you just impersonate the ServiceAccount and ask directly.

---

## Cleanup

Delete all Pods:

```console
kubectl delete pod reader-pod writer-pod
```

Delete the ServiceAccounts:

```console
kubectl delete serviceaccount read-only read-write
```

Delete the Roles:

```console
kubectl delete role pod-reader pod-manager
```

Delete the RoleBindings:

```console
kubectl delete rolebinding read-pods-binding manage-pods-binding
```

---
