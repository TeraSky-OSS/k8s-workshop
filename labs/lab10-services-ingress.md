# Lab 10: Services and Ingress

Pods come and go. Their IP addresses are ephemeral and change every time a Pod is recreated. A Service provides a stable network identity that sits in front of a group of Pods, load-balancing traffic across them regardless of how many there are or which IPs they hold.

This lab covers the three Service types, port-forwarding for local development, and Ingress for HTTP routing.
---

## Step 1: Create a Deployment to expose

Create a 2-replica nginx Deployment. This is the backend you will expose throughout the lab:

```console
kubectl create deployment web --image=nginx --replicas=2
```

Confirm both Pods are running:

```console
kubectl get pods -l app=web
```

---

## Step 2: Expose with ClusterIP

A ClusterIP Service gets a stable virtual IP address that lives inside the cluster. kube-proxy programs routing rules on every node so any Pod in the cluster can reach the Service IP, which then load-balances across the healthy Pods backing it. ClusterIP is the default Service type and is used for any service that should only be reachable within the cluster.

Create a ClusterIP Service:

```console
kubectl expose deployment web --port=80 --target-port=80 --name=web-clusterip
```

Inspect the Service to see its ClusterIP:

```console
kubectl get service web-clusterip
```

Example output:

```console
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
web-clusterip    ClusterIP   10.43.x.x       <none>        80/TCP    5s
```

That `CLUSTER-IP` is stable; it will not change as long as the Service exists.

Now test connectivity from inside the cluster. Run a temporary busybox Pod and attach a shell. The `--rm` flag deletes the Pod when you exit:

```console
kubectl run test-pod --image=busybox --rm -it -- sh
```

Once inside the shell, reach the Service by its short name:

```console
wget -qO- http://web-clusterip
```

You should get the nginx welcome page. CoreDNS resolves `web-clusterip` to the Service's ClusterIP within the `default` namespace. Now try the fully qualified domain name, which works from any namespace in the cluster:

```console
wget -qO- http://web-clusterip.default.svc.cluster.local
```

The FQDN format is `<service>.<namespace>.svc.cluster.local`. This is what inter-service communication uses when services live in different namespaces.

Exit the shell:

```console
exit
```

Delete the ClusterIP Service:

```console
kubectl delete service web-clusterip
```

---

## Step 3: Expose with NodePort

A NodePort Service extends ClusterIP: it does everything ClusterIP does, but also opens a port in the range 30000-32767 on every node in the cluster. Any external traffic hitting `<any-node-ip>:<node-port>` is forwarded into the cluster and load-balanced across the Pods.

NodePort is simple but has limitations: the high port range is awkward to expose publicly, and the port is the same on every node.

Create a NodePort Service:

```console
kubectl expose deployment web --port=80 --target-port=80 --type=NodePort --name=web-nodeport
```

Inspect the Service to see which high port was assigned:

```console
kubectl get service web-nodeport
```

Example output:

```console
NAME           TYPE       CLUSTER-IP      PORT(S)        AGE
web-nodeport   NodePort   10.96.x.x       80:3xxxx/TCP   5s
```

The `80:3xxxx/TCP` format means port 80 inside the cluster maps to the NodePort on the outside. Note the high port number.

Find a node's internal IP to construct the URL:

```console
kubectl get nodes -o wide
```

Pick any node's `INTERNAL-IP` value and combine it with the NodePort:

```console
curl http://<node-ip>:<node-port>
```

You should get the nginx welcome page.

Delete the NodePort Service:

```console
kubectl delete service web-nodeport
```

---

## Step 4: Use port-forward for local access

`kubectl port-forward` tunnels a local port on your machine through the Kubernetes API server to a Pod (or Deployment). It is a development tool, not a networking construct. No Service is created; the API server opens a connection on your behalf.

Start the tunnel:

```console
kubectl port-forward deployment/web 8080:80
```

Example output:

```console
Forwarding from 127.0.0.1:8080 -> 80
```

The terminal is now occupied by the tunnel. Open a second terminal and test it:

```console
curl http://localhost:8080
```

You should get the nginx welcome page. Go back to the first terminal and press Ctrl+C to stop the tunnel.

---

## Step 5: Create an Ingress

An Ingress is an HTTP router. It is a Kubernetes object that holds routing rules (path-based, host-based), but it does nothing on its own. An Ingress controller (in this cluster, nginx-ingress) watches for Ingress objects and configures itself accordingly.

First check that the Ingress controller is running:

```console
kubectl get pods -n ingress-nginx
```

At least one Pod should be in `Running` state. Create a second Deployment alongside the existing `web`:

```console
kubectl create deployment web2 --image=nginx
```

Expose both Deployments as ClusterIP Services:

```console
kubectl expose deployment web --port=80 --name=web-svc
```

```console
kubectl expose deployment web2 --port=80 --name=web2-svc
```

Open a new file in nano:

```console
nano ingress.yaml
```

Type or paste the following content:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /web
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 80
          - path: /web2
            pathType: Prefix
            backend:
              service:
                name: web2-svc
                port:
                  number: 80
```

The `rewrite-target: /` annotation strips the path prefix before the request reaches the backend. Without it, nginx inside the Pod would receive a request for `/web` which it does not know how to serve.

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the Ingress:

```console
kubectl apply -f ingress.yaml
```

Check the Ingress was created and received an address:

```console
kubectl get ingress
```

The `ADDRESS` field may take up to a minute to be populated with the Ingress controller's load balancer IP. Traffic can already be sent while you wait:

```console
# immediately after creating the Ingress:
NAME           CLASS   HOSTS   ADDRESS   PORTS   AGE
demo-ingress   nginx   *                 80      29s

# after ~1 minute:
NAME           CLASS   HOSTS   ADDRESS                                       PORTS   AGE
demo-ingress   nginx   *       172.18.0.2,172.18.0.3,172.18.0.4,172.18.0.5   80      35s
```

Test both paths:

```console
curl http://demo.k8s-workshop.com/web
```

```console
curl http://demo.k8s-workshop.com/web2
```

Both should return the nginx welcome page. The path in the URL determines which Service the Ingress routes to.

---

## Step 6: Virtual host routing

Path-based routing (Step 5) splits traffic by URL path. Virtual host routing splits by the `Host` header. A browser or curl client sends a `Host` header with every HTTP request; the Ingress controller reads it and routes to the appropriate backend. This is how `api.example.com` and `web.example.com` can share one IP and one Ingress controller.

Create three lightweight Deployments to simulate independent services:

```console
kubectl create deployment blue --image=nginx --replicas=1
```

```console
kubectl create deployment green --image=nginx --replicas=1
```

```console
kubectl create deployment yellow --image=nginx --replicas=1
```

Expose each as a ClusterIP Service:

```console
kubectl expose deployment blue --port=80 --name=blue-svc
```

```console
kubectl expose deployment green --port=80 --name=green-svc
```

```console
kubectl expose deployment yellow --port=80 --name=yellow-svc
```

Customise each nginx Pod's default page so you can tell responses apart. This writes a single word into the HTML file that nginx serves:

```console
kubectl exec deployment/blue -- sh -c 'echo blue > /usr/share/nginx/html/index.html'
```

```console
kubectl exec deployment/green -- sh -c 'echo green > /usr/share/nginx/html/index.html'
```

```console
kubectl exec deployment/yellow -- sh -c 'echo yellow > /usr/share/nginx/html/index.html'
```

Open a new file in nano:

```console
nano vhost-ingress.yaml
```

Type or paste the following content:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vhost-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: blue.k8s-workshop.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: blue-svc
                port:
                  number: 80
    - host: green.k8s-workshop.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: green-svc
                port:
                  number: 80
    - host: yellow.k8s-workshop.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: yellow-svc
                port:
                  number: 80
```

Save: Ctrl+O, then Enter. Exit: Ctrl+X.

Apply the Ingress:

```console
kubectl apply -f vhost-ingress.yaml
```

Confirm both Ingresses exist:

```console
kubectl get ingress
```

The `ADDRESS` field may take up to a minute to be populated with the Ingress controller's load balancer IP. Traffic can already be sent while you wait.

Test each virtual host. The `*.k8s-workshop.com` wildcard is already configured in DNS to point to this machine, so no local hosts file changes are needed:

```console
curl http://blue.k8s-workshop.com
```

Example output:

```console
blue
```

```console
curl http://green.k8s-workshop.com
```

Example output:

```console
green
```

```console
curl http://yellow.k8s-workshop.com
```

Example output:

```console
yellow
```

Each request gets a different response based purely on the `Host` header. The same IP, the same Ingress controller, three different backends.

---

## Cleanup

Delete all Deployments:

```console
kubectl delete deployment web web2 blue green yellow
```

Delete all Services:

```console
kubectl delete service web-svc web2-svc blue-svc green-svc yellow-svc
```

Delete both Ingresses:

```console
kubectl delete ingress demo-ingress vhost-ingress
```

---
