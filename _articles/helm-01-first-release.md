---
layout: post
title: "Your First Helm Release"
series: "Helm"
series_url: /helm-series/
part: 1
date: 2026-01-12
author: Quan Huynh
subtitle: "Install Helm 4, add a chart repository, and deploy your first application to Kubernetes with a single command."
tags: [kubernetes, helm, devops]
image: /assets/images/posts/helm-01-first-release/cover.svg
---

In this chapter we get our hands dirty: install Helm, deploy a real application from a
public chart, and learn the handful of CLI commands you'll use every day.

## Installing Helm

On macOS, Linux, or WSL the easiest way is Homebrew:

```bash
brew install helm
```

Other options are the official install script or your system package manager — see the
[Helm installation docs](https://helm.sh/docs/intro/install/). Verify the install:

```bash
helm version
```

```
version.BuildInfo{Version:"v4.0.0", GitCommit:"...", GoVersion:"go1.24"}
```

Helm talks to whatever cluster your current `kubectl` context points at — it reads the
same `~/.kube/config`. So make sure you can reach a cluster first:

```bash
kubectl get nodes
```

Any cluster works: a local one like **kind**, **k3d**, or **minikube**, or a managed
cluster like EKS/GKE/AKS.

## Installing a chart

Let's deploy something real. We'll use the community
[CloudPirates Redis chart](https://artifacthub.io/packages/helm/cloudpirates-redis/redis).
It's published as an **OCI chart** — stored in a container registry, the way Helm 4
prefers — so we can install it straight from its URL, with no `helm repo add` step:

```bash
helm install my-redis oci://registry-1.docker.io/cloudpirates/redis --version 0.33.0
```

That's it — Helm pulled the chart, rendered its templates, and applied them to your
cluster. The output shows the release name, namespace, revision, and the chart's notes
(often with handy next steps).

> **OCI vs classic repos.** Older charts live in HTTP *chart repositories* that you add
> with `helm repo add <name> <url>` and then install as `<name>/<chart>`. Helm 4 makes
> **OCI registries** first-class, so many charts (like this one) install directly from an
> `oci://` URL. We'll use OCI throughout — just know both styles exist.

`my-redis` is the **release name**. Check what you just created:

```bash
kubectl get pods,svc -l app.kubernetes.io/instance=my-redis
```

## The core CLI commands

These are the commands you'll reach for constantly.

**List your releases:**

```bash
helm list
```

```
NAME      NAMESPACE  REVISION  UPDATED   STATUS    CHART         APP VERSION
my-redis  default    1         ...       deployed  redis-0.33.0  8.8.0
```

**Inspect a release** — see the values it was installed with:

```bash
helm get values my-redis
```

**Upgrade** — change something and re-apply. For example, switch to a replicated setup
with three instances:

```bash
helm upgrade my-redis oci://registry-1.docker.io/cloudpirates/redis --version 0.33.0 \
  --set architecture=replication --set replicaCount=3
```

Notice the revision in `helm list` bumps to `2`. Helm kept the old revision so you can
undo.

**Roll back** to the previous revision:

```bash
helm rollback my-redis 1
```

**Uninstall** the release and everything it created:

```bash
helm uninstall my-redis
```

## `--set` vs values files

We passed `--set` flags on the command line. That's fine for one or two tweaks, but for
anything real you'll keep your settings in a **values file** — a plain YAML file of
overrides:

```yaml
# my-values.yaml
architecture: replication
replicaCount: 3
```

```bash
helm install my-redis oci://registry-1.docker.io/cloudpirates/redis --version 0.33.0 -f my-values.yaml
```

Values files are checked into Git, reviewed in pull requests, and reused across
environments. We'll lean on them heavily for the rest of the series.

## Preview before you apply

Two commands let you see exactly what Helm *would* do before touching the cluster:

```bash
# Render templates locally and print the manifests
helm template my-redis oci://registry-1.docker.io/cloudpirates/redis --version 0.33.0 -f my-values.yaml

# Do a full server-side dry run (also runs validation)
helm install my-redis oci://registry-1.docker.io/cloudpirates/redis --version 0.33.0 -f my-values.yaml --dry-run
```

Get in the habit of running `helm template` when something looks off — it shows the
final YAML with all values substituted.

## Where do releases live?

Helm stores each release's history as Secrets in the release's namespace (you'll see
names like `sh.helm.release.v1.my-redis.v1`). That's how `helm rollback` and
`helm history` work — the state lives in the cluster, not on your laptop.

```bash
helm history my-redis
```

The example values file for this chapter is in
[01-first-release](https://github.com/VersusControl/devops-vn-blog/tree/main/_resource/helm-series/01-first-release).

In the [next chapter](/helm-02-anatomy-of-a-chart/) we'll stop *using* other people's
charts and start *building* our own — by dissecting exactly what's inside a chart.
