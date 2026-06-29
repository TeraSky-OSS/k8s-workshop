# Lab 19: Install Istio and Explore the Mesh

Istio is a service mesh: it adds a proxy container to every pod in labelled namespaces and intercepts all traffic between them. In this lab you'll install Istio on the cluster, enable sidecar injection, deploy two services, and verify that mutual TLS (mTLS) is working automatically.
---

## Step 1: Verify istioctl is available

`istioctl` is pre-installed on your lab machine:

```console
istioctl version
```

Example output:

```console
Istio is not present in the cluster: no running Istio pods in namespace "istio-system"
client version: 1.30.1
```

The client is ready. The control plane does not exist yet.

---

## Step 2: Install Istio

Install Istio using the `minimal` profile. This deploys only `istiod`, the control plane. It is the right choice for a single-cluster lab: lightweight, no egress gateway, just the core.

```console
istioctl install --set profile=minimal -y
```

Example output:

```console
        |\          
        | \         
        |  \        
        |   \       
      /||    \      
     / ||     \     
    /  ||      \    
   /   ||       \   
  /    ||        \  
 /     ||         \ 
/______||__________\
____________________
  \__       _____/  
     \_____/        

✔ Istio core installed ⛵️
✔ Istiod installed 🧠
✔ Installation complete
```

Verify the control plane is running:

```console
kubectl get pods -n istio-system
```

Example output (one pod, READY 1/1):

```console
NAME                      READY   STATUS    RESTARTS   AGE
istiod-xxxxxxxxxx-xxxxx   1/1     Running   0          60s
```

Confirm `istioctl` sees the running control plane:

```console
istioctl version
```

Example output:


```console
client version: 1.30.1
control plane version: 1.30.1
data plane version: none
```

---

## Step 3: Create a namespace with sidecar injection enabled

Sidecar injection is opt-in per namespace. Label a namespace and every new pod created in it will automatically get an Envoy proxy container injected.

```console
kubectl create namespace mesh-demo
kubectl label namespace mesh-demo istio-injection=enabled
```

Verify the label is applied:

```console
kubectl get namespace mesh-demo --show-labels
```

Example output includes `istio-injection=enabled`.

---

## Step 4: Deploy httpbin (an HTTP test service)

`httpbin` is a standard HTTP testing utility that returns detailed information about requests. It is useful for inspecting headers, verifying TLS, and testing routing rules.

```console
kubectl apply -n mesh-demo -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
  template:
    metadata:
      labels:
        app: httpbin
    spec:
      containers:
      - name: httpbin
        image: kong/httpbin:0.1.0
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
spec:
  selector:
    app: httpbin
  ports:
  - port: 80
    targetPort: 80
EOF
```

---

## Step 5: Deploy a sleep pod (curl client)

This pod has nothing in it except `curl`. You will use it throughout both labs to send requests from inside the mesh.

```console
kubectl apply -n mesh-demo -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
      - name: sleep
        image: curlimages/curl:latest
        command: ["sleep", "infinity"]
EOF
```

Wait for both pods to be ready:

```console
kubectl wait --for=condition=Ready pods --all -n mesh-demo --timeout=90s
```

---

## Step 6: Verify sidecar injection

List the pods and look at the READY column:

```console
kubectl get pods -n mesh-demo
```

Example output (note 2/2, not 1/1):

```console
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-xxxxxxxxxx-xxxxx   2/2     Running   0          60s
sleep-xxxxxxxxxx-xxxxx     2/2     Running   0          45s
```

The `2/2` means two containers are running: your application and the Envoy proxy. On Kubernetes 1.29+, Istio injects the proxy as a **native sidecar** (an init container with `restartPolicy: Always`), so it lives in `.spec.initContainers` rather than `.spec.containers`. The READY count is the reliable signal; the proxy's exact location in the spec is an implementation detail.

Confirm the proxy is registered with the control plane:

```console
istioctl proxy-status -n mesh-demo
```

Example output (one line per pod, all columns SYNCED):

```console
NAME                                   CLUSTER        ISTIOD                      VERSION     SUBSCRIBED TYPES
httpbin-6b9f9b4866-xxnw7.mesh-demo     Kubernetes     istiod-78fddb5dfd-kc589     1.30.1      4 (CDS,LDS,EDS,RDS)
sleep-b79b697d9-78wpz.mesh-demo        Kubernetes     istiod-78fddb5dfd-kc589     1.30.1      4 (CDS,LDS,EDS,RDS)
```

`SYNCED` across all columns means istiod has pushed the full config to the proxy and the proxy is active. This is more authoritative than parsing the pod spec.

---

## Step 7: Test service-to-service connectivity through the mesh

Execute a `curl` from inside the sleep pod to `httpbin`. Traffic will flow through both Envoy sidecars.

```console
kubectl exec -n mesh-demo deploy/sleep -- \
  curl -s http://httpbin/get | head -30
```

Example output: a JSON response from httpbin showing request headers. Look for the `X-Forwarded-Client-Cert` header in the output:

```console
{
  "args": {}, 
  "headers": {
    "Accept": "*/*", 
    "Host": "httpbin", 
    "User-Agent": "curl/8.20.0", 
    "X-Envoy-Attempt-Count": "1", 
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/mesh-demo/sa/default;Hash=9d88932b4f6d74627e00e9c0dfb91438ae236bfae36a69dc5703b90ac86b20ad;Subject=\"\";URI=spiffe://cluster.local/ns/mesh-demo/sa/default"
  }, 
  "origin": "127.0.0.6", 
  "url": "http://httpbin/get"
}
```

That header is set by the receiving Envoy sidecar and carries the SPIFFE identity of the caller. It confirms mTLS is active.

---

## Step 8: Inspect mTLS status

The `X-Forwarded-Client-Cert` header in Step 7 is the proof: mTLS is already active between injected pods. The receiving Envoy sets that header only after a successful mutual TLS handshake.

Check whether any explicit mTLS policy is in place:

```console
kubectl get peerauthentication -A
```

Example output:

```console
No resources found.
```

No `PeerAuthentication` resource means the default applies: **PERMISSIVE** mode. Injected pods negotiate mTLS with each other automatically, but the mesh still accepts plaintext connections from clients without a sidecar. Step 9 changes that.

---

## Step 9: Apply STRICT mTLS

In PERMISSIVE mode, the mesh accepts both mTLS and plaintext connections. In STRICT mode, any connection without a valid certificate is rejected. Apply it to the `mesh-demo` namespace:

```console
kubectl apply -n mesh-demo -f - <<'EOF'
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT
EOF
```

View the mTLS policy:

```console
kubectl get peerauthentication -A
```

Example output:

```console
NAMESPACE   NAME      MODE     AGE
mesh-demo   default   STRICT   8s
```

Run the same connectivity test from the sleep pod again. It should still work because the sleep pod has an Envoy sidecar and presents a certificate:

```console
kubectl exec -n mesh-demo deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" http://httpbin/get
```

Example output: `200`.

---

## Step 10: Confirm a pod without a sidecar is rejected

Create a plain pod outside the mesh (no sidecar injection label on its namespace):

```console
kubectl run probe --image=curlimages/curl:latest \
  --restart=Never -- sleep 3600
```

Wait for it to start:

```console
kubectl wait --for=condition=Ready pod/probe --timeout=60s
```

Try to reach httpbin from this pod. Because it has no Envoy sidecar it cannot present a certificate:

```console
kubectl exec probe -- \
  curl -s --max-time 5 http://httpbin.mesh-demo/get
```

Expected result: the command exits with a non-zero code and no response body. You will see something like:

```console
curl: (56) Recv failure: Connection reset by peer
command terminated with exit code 56
```

Exit code 56 means curl received a TCP reset. Envoy on the httpbin pod accepted the TCP connection, detected that the client could not perform an mTLS handshake (no sidecar, no certificate), and reset the connection. That is the STRICT PeerAuthentication policy working correctly.

Clean up the probe pod:

```console
kubectl delete pod probe
```

---

## Step 11: Clean up

Leave the `mesh-demo` namespace and its workloads running. Lab 19 builds on them.

You can verify the mesh is healthy before moving on:

```console
istioctl proxy-status -n mesh-demo
```

Example output: both `httpbin` and `sleep` listed with `SYNCED` status for all Envoy config types (CDS, LDS, RDS, EDS).

---

## Summary

- Istio's control plane (`istiod`) manages certificates and pushes config to all Envoy sidecars.
- Labelling a namespace with `istio-injection=enabled` is all that's needed to enroll workloads.
- Pods with a sidecar show `2/2` in the READY column.
- mTLS is enforced between injected pods automatically. STRICT mode blocks any connection without a valid certificate.
- The `X-Forwarded-Client-Cert` header and `istioctl authn tls-check` both confirm mTLS is active.
