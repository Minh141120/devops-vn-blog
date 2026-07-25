---
layout: post
title: "Dependencies and Subcharts"
series: "Helm"
series_url: /helm-series/
part: 7
date: 2026-02-23
author: Quan Huynh
subtitle: "Reuse community charts as building blocks — declare dependencies, pass values into subcharts, and manage versions with Chart.lock."
tags: [kubernetes, helm, devops]
image: /assets/images/posts/helm-07-dependencies-and-subcharts/cover.svg
---

Real applications rarely stand alone — they often need a cache, a database, or a message
queue. Rather than write and maintain those charts yourself, Helm lets you pull in an
existing chart as a **dependency**. In this chapter we add a **Redis** cache to Book Info
as a subchart, using the CloudPirates Redis chart.

## Declaring a dependency

Dependencies are listed in `Chart.yaml` under `dependencies`:

```yaml
apiVersion: v2
name: book-info
version: 0.2.0
appVersion: "1.20.3"

dependencies:
  - name: redis
    version: "0.33.0"
    repository: oci://registry-1.docker.io/cloudpirates
    condition: redis.enabled
```

Each dependency has:

- **name** — the chart name in the repository.
- **version** — a SemVer constraint (`0.33.0`, `~0.33.0`, `>=0.33.0 <1.0.0`, …).
- **repository** — where to fetch it (an HTTP repo or an `oci://` registry).
- **condition** — an optional value that toggles the dependency on or off.

## Fetching dependencies

Helm downloads dependencies into the `charts/` folder and writes a `Chart.lock`:

```bash
helm dependency update ./book-info
```

```
Saving 1 charts
Downloading redis from repo oci://registry-1.docker.io/cloudpirates
Deleting outdated charts
```

`Chart.lock` pins the *exact* resolved versions — commit it to Git so every teammate and
every CI run gets identical builds. Later, `helm dependency build` installs precisely
what the lock file records.

## Configuring a subchart

A subchart reads its values from a **key named after the chart** in your parent
`values.yaml`. So to configure the Redis subchart, nest everything under `redis:`:

```yaml
# values.yaml (parent)
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: true
    password: "change-me"
```

Everything under `redis:` is handed to the Redis chart as *its* top-level values. The
`enabled` flag pairs with the `condition: redis.enabled` we set in `Chart.yaml`, so you
can switch Redis off in environments that don't need it:

```yaml
# values-dev.yaml
redis:
  enabled: false
```

## Global values

Sometimes parent and subcharts need to share a value — an image registry or an
environment label. Anything under the special `global:` key is visible to **every**
chart, parent and children alike:

```yaml
global:
  imageRegistry: my-registry.example.com
```

A subchart reads it as `.Values.global.imageRegistry`.

## Inspecting the dependency tree

```bash
helm dependency list ./book-info
```

```
NAME   VERSION  REPOSITORY                               STATUS
redis  0.33.0   oci://registry-1.docker.io/cloudpirates  ok
```

Render everything together to confirm the Redis objects appear alongside Book Info:

```bash
helm template book-info ./book-info | grep -E 'kind: (StatefulSet|Service)'
```

## When to use a subchart vs. a separate release

A subchart is the right choice when the dependency is **part of** your app's lifecycle —
installed, upgraded, and deleted together with it. If a component is shared across many
apps or owned by another team, keep it as its **own release** instead and just point
your app at its Service. Coupling everything into one giant umbrella chart makes upgrades
riskier, so lean toward smaller, focused charts.

The umbrella chart with the Redis dependency is in
[07-dependencies-and-subcharts](https://github.com/VersusControl/devops-vn-blog/tree/main/_resource/helm-series/07-dependencies-and-subcharts).

With dependencies we can compose applications from reusable parts. In the
[next chapter](/helm-08-upgrades-and-rollbacks/) we'll look at the day-2 story — safely
**upgrading, rolling back, and hooking into** a release's lifecycle.
