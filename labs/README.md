# Workshop Labs

Hands-on labs for the Kubernetes workshop. The labs run on dedicated EC2 instances.

## Student connection

```bash
# Connect as your assigned student number
ssh studentX@studentX.ts-k8s-workshop.com
```

## Harbor registry

Harbor runs at `https://$HARBOR_HOST`. Student credentials match their OS username and password. The `workshop` project is public - students can pull each other's images without authentication.

## Docker labs

| Lab | File | Topic |
|-----|------|-------|
| 1 | [lab01-first-containers.md](lab01-first-containers.md) | Pull, run, inspect, manage containers |
| 2 | [lab02-networking.md](lab02-networking.md) | Custom networks and container DNS |
| 3 | [lab03-storage.md](lab03-storage.md) | Volumes and bind mounts |
| 4 | [lab04-dockerfiles-and-multistage.md](lab04-dockerfiles-and-multistage.md) | Dockerfile instructions and multi-stage builds |
| 5 | [lab05-registry.md](lab05-registry.md) | Build, tag, push, and pull images via Harbor |

## Kubernetes labs

| Lab | File | Topic |
| --- | ---- | ----- |
| 6 | [lab06-pods-and-labels.md](lab06-pods-and-labels.md) | Connect to cluster, create namespace, run Pods, labels, annotations |
| 7 | [lab07-deployments.md](lab07-deployments.md) | Create, scale, update, rollout, rollback Deployments |
| 8 | [lab08-pod-config-probes.md](lab08-pod-config-probes.md) | Resource limits, liveness/readiness probes, HPA |
| 9 | [lab09-daemonset-nodes.md](lab09-daemonset-nodes.md) | DaemonSets and node scheduling |
| 10 | [lab10-services-ingress.md](lab10-services-ingress.md) | Services, port-forwarding, and Ingress |
| 11 | [lab11-storage.md](lab11-storage.md) | EmptyDir, PVC, StatefulSet |
| 12 | [lab12-config-and-jobs.md](lab12-config-and-jobs.md) | ConfigMaps, Secrets, Jobs, CronJobs, init containers |
| 13 | [lab13-troubleshooting.md](lab13-troubleshooting.md) | Diagnosing common cluster failure modes |
| 14 | [lab14-rbac.md](lab14-rbac.md) | ServiceAccount, Role, RoleBinding, kubectl auth can-i |
| 15 | [lab15-network-policy.md](lab15-network-policy.md) | NetworkPolicy: default-deny, allow rules, ingress/egress |

## Helm labs

| Lab | File | Topic |
| --- | ---- | ----- |
| 16 | [lab16-helm-install.md](lab16-helm-install.md) | Install Helm, add Bitnami repo, deploy MySQL, inspect release resources |
| 17 | [lab17-helm-revisions.md](lab17-helm-revisions.md) | Create a chart, install, upgrade, history, rollback, custom values file |

## Istio labs

| Lab | File | Topic |
| --- | ---- | ----- |
| 18 | [lab18-istio-install.md](lab18-istio-install.md) | Install Istio, enable sidecar injection, verify mTLS |
| 19 | [lab19-istio-traffic.md](lab19-istio-traffic.md) | Traffic splitting, fault injection, observing mesh behavior |
