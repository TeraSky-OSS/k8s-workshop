# Lab 19: Istio Traffic Management

This lab demonstrates Istio's traffic management features using two versions of a simple echo service. You will split traffic by weight, inject artificial faults, and observe how clients respond to each scenario. All testing is done from the sleep pod inside the mesh with `curl`.
Prerequisites: Lab 18 must be complete. The `mesh-demo` namespace with sidecar injection enabled must exist.

---

## Step 1: Deploy two versions of an echo service

Deploy `echo-v1` and `echo-v2`. Each returns a short text response identifying which version handled the request. Both are exposed under a single Kubernetes Service named `echo`.

```console
kubectl apply -n mesh-demo -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo
      version: v1
  template:
    metadata:
      labels:
        app: echo
        version: v1
    spec:
      containers:
      - name: echo
        image: hashicorp/http-echo:alpine
        args: ["-text=version-1"]
        ports:
        - containerPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-v2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo
      version: v2
  template:
    metadata:
      labels:
        app: echo
        version: v2
    spec:
      containers:
      - name: echo
        image: hashicorp/http-echo:alpine
        args: ["-text=version-2"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  selector:
    app: echo
  ports:
  - port: 80
    targetPort: 5678
EOF
```

Wait for all echo pods to be ready:

```console
kubectl wait --for=condition=Ready pods -l app=echo -n mesh-demo --timeout=90s
```

Verify four pods are running (two per version), each with `2/2` containers:

```console
kubectl get pods -n mesh-demo -l app=echo
```

---

## Step 2: Observe default Kubernetes load balancing

Without any Istio routing rules, Kubernetes distributes requests randomly across all four pods. Send ten requests from the sleep pod and observe the mix of responses:

```console
for i in $(seq 1 10); do
  kubectl exec -n mesh-demo deploy/sleep -- curl -s http://echo/
done
```

You should see a roughly equal mix of `version-1` and `version-2` responses. There is no predictable pattern.

---

## Step 3: Define subsets with a DestinationRule

A `DestinationRule` maps subset names to pod label selectors. This is what the VirtualService will reference when directing traffic.

```console
kubectl apply -n mesh-demo -f - <<'EOF'
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: echo
spec:
  host: echo
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF
```

---

## Step 4: Route all traffic to v1

Apply a VirtualService that sends 100% of traffic to the `v1` subset:

```console
kubectl apply -n mesh-demo -f - <<'EOF'
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: echo
spec:
  hosts:
  - echo
  http:
  - route:
    - destination:
        host: echo
        subset: v1
      weight: 100
EOF
```

Send ten requests and confirm they all go to v1:

```console
for i in $(seq 1 10); do
  kubectl exec -n mesh-demo deploy/sleep -- curl -s http://echo/
done
```

All responses should read `version-1`. v2 pods are still running, but Istio's routing rules send no traffic to them.

---

## Step 5: Shift to a 80/20 split (canary)

Update the VirtualService to send 20% of requests to v2:

```console
kubectl apply -n mesh-demo -f - <<'EOF'
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: echo
spec:
  hosts:
  - echo
  http:
  - route:
    - destination:
        host: echo
        subset: v1
      weight: 80
    - destination:
        host: echo
        subset: v2
      weight: 20
EOF
```

Send 20 requests and count how many go to each version:

```console
kubectl exec -n mesh-demo deploy/sleep -- sh -c '
  v1=0; v2=0
  for i in $(seq 1 20); do
    r=$(curl -s http://echo/)
    case "$r" in
      *version-1*) v1=$((v1+1)) ;;
      *version-2*) v2=$((v2+1)) ;;
    esac
  done
  echo "v1: $v1  v2: $v2"
'
```

Example output: approximately 16 v1 and 4 v2 (80/20 split). The exact numbers vary because percentage-based weighting is probabilistic, but the trend is clear over 20 requests.

---

## Step 6: Inject a delay fault

Istio can delay a percentage of requests without changing any application code. Apply a 5-second delay to 50% of requests to the `echo` service:

```console
kubectl apply -n mesh-demo -f - <<'EOF'
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: echo
spec:
  hosts:
  - echo
  http:
  - fault:
      delay:
        percentage:
          value: 50
        fixedDelay: 5s
    route:
    - destination:
        host: echo
        subset: v1
      weight: 80
    - destination:
        host: echo
        subset: v2
      weight: 20
EOF
```

Send a request and time it. About half will respond immediately; the other half take 5 seconds:

```console
kubectl exec -n mesh-demo deploy/sleep -- sh -c '
  for i in $(seq 1 6); do
    time curl -s http://echo/ > /dev/null
  done
'
```

Look at the elapsed time per request. You should see some complete in under a second and others take around 5 seconds.

---

## Step 7: Inject an HTTP abort fault

Remove the delay and inject an HTTP 503 abort on 30% of requests instead:

```console
kubectl apply -n mesh-demo -f - <<'EOF'
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: echo
spec:
  hosts:
  - echo
  http:
  - fault:
      abort:
        percentage:
          value: 30
        httpStatus: 503
    route:
    - destination:
        host: echo
        subset: v1
      weight: 80
    - destination:
        host: echo
        subset: v2
      weight: 20
EOF
```

Send 10 requests and print the HTTP status code for each:

```console
kubectl exec -n mesh-demo deploy/sleep -- sh -c '
  for i in $(seq 1 10); do
    curl -s -o /dev/null -w "%{http_code}\n" http://echo/
  done
'
```

Example output: a mix of `200` and `503` responses, with roughly 3 out of 10 being 503. This simulates a degraded upstream service and lets you verify whether clients handle the errors correctly.

---

## Step 8: Remove fault injection

Clean up the fault injection and restore normal 80/20 routing:

```console
kubectl apply -n mesh-demo -f - <<'EOF'
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: echo
spec:
  hosts:
  - echo
  http:
  - route:
    - destination:
        host: echo
        subset: v1
      weight: 80
    - destination:
        host: echo
        subset: v2
      weight: 20
EOF
```

Verify requests succeed again with normal latency:

```console
for i in $(seq 1 5); do
  kubectl exec -n mesh-demo deploy/sleep -- curl -s http://echo/
done
```

---

## Step 9: Cut over to v2 (blue-green)

Completing the canary promotion: send all traffic to v2, then remove v1.

```console
kubectl apply -n mesh-demo -f - <<'EOF'
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: echo
spec:
  hosts:
  - echo
  http:
  - route:
    - destination:
        host: echo
        subset: v2
      weight: 100
EOF
```

Confirm all requests now go to v2:

```console
for i in $(seq 1 5); do
  kubectl exec -n mesh-demo deploy/sleep -- curl -s http://echo/
done
```

All responses should read `version-2`. Scale down v1 now that it is receiving no traffic:

```console
kubectl scale deploy/echo-v1 -n mesh-demo --replicas=0
```

---

## Step 10: Clean up

Remove all Istio routing config and the lab namespace:

```console
kubectl delete namespace mesh-demo
```

This removes the Deployments, Services, VirtualServices, DestinationRules, and PeerAuthentication in one command.

Verify the namespace is gone:

```console
kubectl get namespace mesh-demo
```

Example output: `Error from server (NotFound): namespaces "mesh-demo" not found`.

---

## Summary

- A `DestinationRule` maps subset names to pod labels.
- A `VirtualService` controls where traffic goes: by weight, by header, or by path.
- Traffic weights are changed at runtime by updating the VirtualService. No pod restarts, no rollout.
- Fault injection (delay and abort) tests how clients behave under degraded conditions, with no application code changes.
- A blue-green cutover is a weight change from 100/0 to 0/100, then a scale-down of the old version.
