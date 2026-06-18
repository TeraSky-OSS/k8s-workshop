# Lab 17: Install and Explore a Helm Chart

Helm is Kubernetes' package manager. A chart is a package that bundles all the YAML manifests needed to deploy an application, along with default values and templates. Helm renders the templates with your values, sends the result to the Kubernetes API, and tracks what it deployed as a named release. This gives you upgrades, rollbacks, and a consistent way to manage third-party software.

This lab adds a chart repository, installs a real application, and inspects everything Helm created.
---

## Step 1: Verify Helm is installed

Helm is pre-installed on your lab machine. Confirm it is available:

```console
helm version
```

Example output (version numbers may differ):

```console
version.BuildInfo{Version:"v4.2.0", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.24.x"}
```

---

## Step 2: Add the podinfo chart repository

Chart repositories are like package registries: a URL that hosts an index of available charts. You add a repository once, then install any chart from it by name.

`podinfo` is a lightweight Go microservice built specifically for Kubernetes demos. Its chart is small, has no external dependencies, and deploys in seconds.

Add the repository:

```console
helm repo add podinfo https://stefanprodan.github.io/podinfo
```

Example output:

```console
"podinfo" has been added to your repositories
```

Fetch the latest chart index from all configured repositories. This is equivalent to `apt update`: it refreshes your local cache of what is available:

```console
helm repo update
```

Example output:

```console
...Successfully got an update from the "podinfo" chart repository
Update Complete. Happy Helming!
```

List all configured repositories to confirm yours is registered:

```console
helm repo list
```

---

## Step 3: Browse the chart

Search your local repository index for podinfo charts:

```console
helm search repo podinfo
```

Example output:

```console
NAME              CHART VERSION   APP VERSION   DESCRIPTION
podinfo/podinfo   6.x.x           6.x.x         Podinfo Helm chart for Kubernetes
```

Before installing, look at the chart's default values. This is the chart's configuration interface: every value listed here can be overridden at install time:

```console
helm show values podinfo/podinfo
```

Scroll through the output. Notice the `ui` section near the bottom, which exposes `color` and `message` fields. You will override these in the upgrade step.

---

## Step 4: Install the chart

`helm install` creates a release: a named instance of a chart running in your cluster. The name `my-podinfo` is the release name; you use it to identify this specific installation in all subsequent Helm commands.

Install podinfo with ingress enabled so you can reach the UI in a browser:

```console
helm install my-podinfo podinfo/podinfo \
  --set ingress.enabled=true \
  --set ingress.className=nginx \
  --set "ingress.hosts[0].host=podinfo.k8s-workshop.com" \
  --set "ingress.hosts[0].paths[0].path=/" \
  --set "ingress.hosts[0].paths[0].pathType=Prefix"
```

`--set` overrides individual values from the chart's `values.yaml`. Any value not explicitly set keeps its chart default.

Wait for the Pod to be ready:

```console
kubectl get pods -w
```

Press Ctrl+C once the Pod shows `Running`.

Test the application:

```console
curl http://podinfo.k8s-workshop.com
```

You should see a JSON response from podinfo describing its environment.

---

## Step 5: Explore what Helm created

`kubectl get all` finds most resources by label selector:

```console
kubectl get all --selector=app.kubernetes.io/name=my-podinfo
```

Secrets are not included in `get all`. Helm stores its own release state as Secrets in the namespace. List them:

```console
kubectl get secret --selector=owner=helm
```

The Secret named `sh.helm.release.v1.my-podinfo.v1` is Helm's record of what was deployed at revision 1. Each upgrade creates a new versioned Secret. Helm uses these to implement rollbacks.

Check the overall release status:

```console
helm list
```

Get more detail about this specific release, including its post-install notes:

```console
helm status my-podinfo
```

`helm list` shows the release name, namespace, revision, and deploy status. `helm status` adds the chart's NOTES.txt output, which typically contains access instructions.

---

## Step 6: Inspect the rendered manifests

Helm works by rendering Go templates with your values before sending anything to Kubernetes. You can see the exact YAML that was sent:

```console
helm get manifest my-podinfo
```

This shows every resource Helm applied: Deployment, Service, HorizontalPodAutoscaler, ServiceAccount. The template expressions (like `{{ .Release.Name }}`) have already been resolved into actual values. This is the output you compare against when debugging discrepancies between what Helm thinks it deployed and what Kubernetes actually has.

Check which values were used for this release:

```console
helm get values my-podinfo
```

This shows only the values you explicitly set (the ingress overrides).

Now add the `--all` flag to see every value including chart defaults:

```console
helm get values my-podinfo --all
```

The full set shows every configurable field and what it resolved to. Useful for understanding what a chart actually configures by default.

---

## Step 7: Upgrade the release

`helm upgrade` applies a new set of values to an existing release. The revision counter increments.

One important detail: Helm does not merge your new values with the old ones by default. It replaces the entire values set with what you provide. If you do not re-pass the ingress values from the install step, they will revert to their chart defaults. Always carry forward the values you care about.

Upgrade the release to change the UI colour and message:

```console
helm upgrade my-podinfo podinfo/podinfo \
  --set ingress.enabled=true \
  --set ingress.className=nginx \
  --set "ingress.hosts[0].host=podinfo.k8s-workshop.com" \
  --set "ingress.hosts[0].paths[0].path=/" \
  --set "ingress.hosts[0].paths[0].pathType=Prefix" \
  --set ui.color="#00b3a4" \
  --set ui.message="Upgraded with Helm"
```

Check that the revision counter incremented:

```console
helm list
```

Example output:

```console
NAME         NAMESPACE   REVISION   STATUS     CHART
my-podinfo   default     2          deployed   podinfo-6.x.x
```

Confirm the UI reflects the changes:

```console
curl http://podinfo.k8s-workshop.com
```

Look for `"color":"#00b3a4"` and `"message":"Upgraded with Helm"` in the JSON response.

View the full release history to see both revisions:

```console
helm history my-podinfo
```

Example output:

```console
REVISION   STATUS       CHART           DESCRIPTION
1          superseded   podinfo-6.x.x   Install complete
2          deployed     podinfo-6.x.x   Upgrade complete
```

Revision 1 is `superseded`: replaced by revision 2, but still stored as a Secret so it can be rolled back to.

---

## Step 8: Check Helm environment

```console
helm env
```

This shows the directories Helm uses for its local cache, config, and repository data. Useful when debugging issues with stale index files or missing plugins.

---

## Cleanup

Uninstall the release. This deletes all Kubernetes resources the release created, as well as the Helm state Secrets:

```console
helm uninstall my-podinfo
```

Example output:

```console
release "my-podinfo" uninstalled
```

Verify the resources are gone:

```console
kubectl get all --selector=app.kubernetes.io/name=my-podinfo
```

Example output:

```console
No resources found in default namespace.
```

---
