# Lab 15: Network Policy

By default, every Pod in a Kubernetes cluster can talk to every other Pod. There are no network boundaries between workloads. NetworkPolicy objects let you change that: you define rules that control which Pods can send traffic to which other Pods.

There is one important prerequisite: NetworkPolicy enforcement is not built into Kubernetes itself. It is the job of the CNI (Container Network Interface) plugin. This cluster uses Canal (Flannel + Calico), which enforces policies. With a CNI plugin that does not support NetworkPolicy, such as plain Flannel, you can create NetworkPolicy objects, but they will have no effect.
---

## Step 1: Baseline - traffic works without any policy

Before adding any restrictions, confirm that traffic flows freely between Pods. This is the starting state.

Run a web server and expose it within the namespace:

```console
kubectl run web --image=nginx --labels=app=web --port=80
```

```console
kubectl expose pod web --port=80 --name=web-svc
```

From a temporary client Pod, confirm the server is reachable. The `--rm` flag cleans up the Pod when you exit:

```console
kubectl run client --image=busybox --rm -it -- sh
```

Once inside the shell, make a request to the web server:

```console
wget -qO- --timeout=3 http://web-svc
```

Example output: the nginx welcome page HTML.

Exit the shell:

```console
exit
```

Traffic flows freely because no NetworkPolicy has been applied yet. With no policy, there is no restriction.

---

## Step 2: Deny all ingress to the web Pod

A NetworkPolicy that declares `policyTypes: [Ingress]` with an empty `ingress: []` list means: match these Pods, and block all inbound traffic. There are no allowed sources, so nothing gets through.

Open a new file in nano:

```console
nano web-deny-all.yaml
```

Type or paste the following content:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-deny-all
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress: []
```

The `podSelector` selects which Pods this policy applies to. Only Pods with `app=web` are affected. Other Pods in the namespace are untouched.

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the policy:

```console
kubectl apply -f web-deny-all.yaml
```

Now test again from a client Pod:

```console
kubectl run client --image=busybox --rm -it -- sh
```

Once inside the shell:

```console
wget -qO- --timeout=3 http://web-svc
```

Example output:

```console
wget: download timed out
```

Exit the shell:

```console
exit
```

The request timed out. The policy took effect immediately after being applied.

---

## Step 3: Allow traffic only from Pods with a specific label

NetworkPolicy rules are additive: multiple policies for the same `podSelector` are ORed together. You do not need to modify the deny-all policy. Instead, you add a separate allow policy for the specific traffic you want to permit.

This policy allows ingress to `app=web` Pods, but only from Pods that carry the label `role=frontend`.

Delete the current deny-all policy first (so you can see the transition clearly):

```console
kubectl delete networkpolicy web-deny-all
```

Open a new file in nano:

```console
nano web-allow-frontend.yaml
```

Type or paste the following content:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-allow-frontend
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
```

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the policy:

```console
kubectl apply -f web-allow-frontend.yaml
```

Test from a client Pod **without** the `role=frontend` label. This should be blocked:

```console
kubectl run no-label --image=busybox --rm -it -- sh
```

Once inside:

```console
wget -qO- --timeout=3 http://web-svc
```

Example output:

```console
wget: download timed out
```

Exit:

```console
exit
```

Now test from a Pod **with** the `role=frontend` label. This should succeed:

```console
kubectl run frontend --image=busybox --labels=role=frontend --rm -it -- sh
```

Once inside:

```console
wget -qO- --timeout=3 http://web-svc
```

Example output: the nginx welcome page HTML.

Exit:

```console
exit
```

The label on the client Pod is what grants access. Remove the label (or run without it) and you are blocked.

---

## Step 4: Namespace-wide default deny

The recommended security baseline for any namespace that runs sensitive workloads is a default-deny policy: block all ingress by default, then add explicit allow policies for each service that needs to receive traffic.

An empty `podSelector: {}` (no `matchLabels`) matches every Pod in the namespace. When combined with `policyTypes: [Ingress]` and no `ingress` rules, it blocks all inbound traffic to all Pods.

Open a new file in nano:

```console
nano default-deny-all.yaml
```

Type or paste the following content:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the baseline policy:

```console
kubectl apply -f default-deny-all.yaml
```

Verify the current state. You now have two policies in the namespace:

```console
kubectl get networkpolicy
```

Now test: a `role=frontend` Pod should still be able to reach the web server because `web-allow-frontend` is still active. NetworkPolicy rules for the same destination are ORed: the Ingress to `app=web` from `role=frontend` is explicitly allowed, which takes precedence over the default deny.

```console
kubectl run frontend --image=busybox --labels=role=frontend --rm -it -- sh
```

Once inside:

```console
wget -qO- --timeout=3 http://web-svc
```

Example output: the nginx welcome page HTML.

Exit:

```console
exit
```

Access still works for `role=frontend`. Every other Pod in the namespace is now blocked from reaching `app=web`, and blocked from reaching each other. New services added to this namespace are automatically protected until you add an explicit allow for them.

---

## Step 5: Clean up

Delete the web Pod and its Service:

```console
kubectl delete pod web
```

```console
kubectl delete service web-svc
```

Delete both NetworkPolicies:

```console
kubectl delete networkpolicy web-allow-frontend default-deny-all
```

---
