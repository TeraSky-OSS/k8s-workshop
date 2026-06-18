# Lab 17: Revisions, Rollbacks, and Value Overrides

Every time you install or upgrade a Helm release, Helm increments a revision counter and stores a complete snapshot of what was deployed. This makes rollbacks instant: Helm just re-applies an older snapshot without needing any external state.

This lab walks through the full lifecycle of a chart you create yourself: install, upgrade, inspect history, roll back, and override defaults with a values file.
---

## Step 1: Scaffold a new chart

`helm create` generates a complete, working chart with sensible defaults. It is the standard starting point for any new chart.

```console
helm create testchart
```

List the generated files:

```console
ls testchart/
```

Example output:

```console
Chart.yaml  charts  templates  values.yaml
```

- `Chart.yaml` is the chart's metadata: name, version, description.
- `values.yaml` is the chart's default configuration. Every value here can be overridden at install or upgrade time.
- `templates/` holds the Go template files that Helm renders into Kubernetes manifests.
- `charts/` holds subcharts (dependencies). It is empty by default.

Inspect the top of the default values file:

```console
cat testchart/values.yaml | head -20
```

Notice `replicaCount: 1` near the top. The default chart deploys a single nginx Pod. The rest of the values configure the Service, Ingress, and resource limits.

---

## Step 2: Install the chart

Install the chart as a release named `demo`:

```console
helm install demo testchart
```

Example output (abbreviated):

```console
NAME: demo
LAST DEPLOYED: ...
STATUS: deployed
REVISION: 1
```

`REVISION: 1` is the starting point. Every upgrade increments this number.

Check the Kubernetes resources Helm created:

```console
kubectl get deploy,pods
```

You should see one Deployment and one running Pod. The names follow the pattern `<release-name>-<chart-name>`, so `demo-testchart`.

List all Helm releases in the current namespace:

```console
helm list
```

---

## Step 3: Upgrade - change replica count

The values file is the chart's configuration. Editing it and running `helm upgrade` is the declarative way to change a release.

Open `testchart/values.yaml` in a text editor:

```console
nano testchart/values.yaml
```

Find `replicaCount: 1` and change it to `replicaCount: 3`.

Save the file: press Ctrl+O, then Enter. Exit: Ctrl+X.

Run the upgrade:

```console
helm upgrade demo testchart
```

Example output:

```console
Release "demo" has been upgraded. Happy Helming!
NAME: demo
REVISION: 2
```

The revision incremented from 1 to 2.

Watch the two new Pods start:

```console
kubectl get pods -w
```

Press Ctrl+C once all three Pods are Running.

---

## Step 4: Inspect revision history

Every revision is stored as a Secret in the namespace. This is what makes rollbacks possible without any external state or database.

View the history:

```console
helm history demo
```

Example output:

```console
REVISION  UPDATED             STATUS      CHART            APP VERSION  DESCRIPTION
1         <timestamp>         superseded  testchart-0.1.0  1.16.0       install complete
2         <timestamp>         deployed    testchart-0.1.0  1.16.0       upgrade complete
```

Revision 1 is `superseded`: it has been replaced by revision 2, but the snapshot is still there. Check the Secrets Helm uses to store this:

```console
kubectl get secret | grep helm
```

You should see two Secrets: `sh.helm.release.v1.demo.v1` and `sh.helm.release.v1.demo.v2`. Each contains a complete record of the chart templates, the values used, and the rendered manifests for that revision.

---

## Step 5: Roll back to revision 1

Rolling back to revision 1 will reduce the replica count from 3 back to 1 (since revision 1 used `replicaCount: 1`).

```console
helm rollback demo 1
```

Example output:

```console
Rollback was a success! Happy Helming.
```

Verify the Pod count dropped back to 1:

```console
kubectl get pods
```

Check the history again:

```console
helm history demo
```

Example output:

```console
REVISION  UPDATED             STATUS      CHART            APP VERSION  DESCRIPTION
1         <timestamp>         superseded  testchart-0.1.0  1.16.0       install complete
2         <timestamp>         superseded  testchart-0.1.0  1.16.0       upgrade complete
3         <timestamp>         deployed    testchart-0.1.0  1.16.0       rollback to 1
```

Rollback created revision 3, not a reset to revision 1. Helm never rewrites history. Every change, including rollbacks, is recorded as a new revision. This is an important audit property: you can always see the sequence of changes that brought the release to its current state.

---

## Step 6: Override values with a custom file

So far you have been editing the chart's `values.yaml` directly. That works for a chart you own, but it couples your configuration to the chart source. The cleaner approach is a separate values file (`-f my-values.yaml`) that you pass at install or upgrade time. The chart's defaults remain untouched; your overrides are version-controlled separately.

Before writing the override file, reset `testchart/values.yaml` back to `replicaCount: 1`:

```console
nano testchart/values.yaml
```

Change `replicaCount: 3` back to `replicaCount: 1`. Save and exit.

Create a separate override file:

```console
nano my-values.yaml
```

Add the following content:

```yaml
replicaCount: 5
```

Save and exit.

Install a second release named `demo2` from the same chart, but with your override file:

```console
helm install demo2 testchart -f my-values.yaml
```

Check what is running now:

```console
kubectl get deploy,pods
```

You should see `demo` (1 replica) and `demo2` (5 replicas) running side by side from the same chart source. The `-f` flag layered your overrides on top of the chart defaults, with your file winning for any key that appears in both.

List both releases:

```console
helm list
```

---

## Step 7: Override the release name in resource names

Helm names resources `<release-name>-<chart-name>` by default. Sometimes the generated name is too long or does not match the naming convention you want. `fullnameOverride` replaces the entire computed name used for Kubernetes resource names. The Helm release name (`helm list`) is unaffected.

Open the override file:

```console
nano my-values.yaml
```

The file should now contain:

```yaml
replicaCount: 5
fullnameOverride: mywebapp
```

Save and exit.

Upgrade `demo2` with the updated values file:

```console
helm upgrade demo2 testchart -f my-values.yaml
```

Check the Deployments and Pods:

```console
kubectl get deploy,pods
```

The `demo2` Pods are now named `mywebapp-<hash>` instead of `demo2-testchart-<hash>`. Verify the release name in Helm's view has not changed:

```console
helm list
```

`demo2` still shows as the release name. The release name and the Kubernetes resource names are separate concepts.

---

## Cleanup

Uninstall both releases:

```console
helm uninstall demo demo2
```

Example output:

```console
release "demo" uninstalled
release "demo2" uninstalled
```

Verify all resources are gone:

```console
kubectl get all
```

Verify Helm has no active releases:

```console
helm list
```

Both should return no resources in your namespace.

---
