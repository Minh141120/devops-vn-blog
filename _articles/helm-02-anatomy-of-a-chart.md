---
layout: post
title: "Anatomy of a Helm Chart"
series: "Helm"
series_url: /helm-series/
part: 2
date: 2026-01-19
author: Quan Huynh
subtitle: "Run helm create and tour every file in a chart — Chart.yaml, values.yaml, templates, and helpers — so you know exactly what each piece does."
tags: [kubernetes, helm, devops]
image: /assets/images/posts/helm-02-anatomy-of-a-chart/cover.svg
---

Now that we can install charts, let's open one up. In this chapter we scaffold a brand
new chart and walk through every file, so the rest of the series has a solid
foundation.

## Scaffold a chart

Helm can generate a working starter chart for you:

```bash
helm create demo
```

This creates a `demo/` folder:

```
demo/
├── Chart.yaml
├── values.yaml
├── charts/
├── templates/
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   └── tests/
└── .helmignore
```

Let's go through the important pieces.

## `Chart.yaml` — the chart's identity card

`Chart.yaml` holds metadata about the chart itself:

```yaml
apiVersion: v2
name: demo
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
```

- **apiVersion: v2** — the modern chart format (used by Helm 3 and 4).
- **name / description** — what the chart is.
- **type** — `application` (something you deploy) or `library` (shared template helpers,
  not installable on its own).
- **version** — the version of *the chart*, bumped when you change the chart. Uses
  SemVer.
- **appVersion** — the version of *the app inside* the chart (e.g. the image tag). This
  is just metadata; it doesn't have to be SemVer.

Keep the two versions straight: `version` is about the packaging, `appVersion` is about
the software being packaged.

## `values.yaml` — the default configuration

`values.yaml` is the single source of default settings. Everything a user might want to
change lives here:

```yaml
replicaCount: 1

image:
  repository: nginx
  tag: ""
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
```

Templates read these values (we'll see how next chapter). Users override them with
`-f my-values.yaml` or `--set`, but they never have to edit the templates.

## `templates/` — the manifests, parameterized

This is where the real work happens. Each file in `templates/` is a Kubernetes manifest
with placeholders. A trimmed `deployment.yaml` looks like this:

{% raw %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-demo
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: demo
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```
{% endraw %}

Anything inside `{% raw %}{{ }}{% endraw %}` is a **template action** that Helm evaluates
at install time. `.Values` reaches into `values.yaml`; `.Release` is information about
this particular install. We'll dig into all of these in the next chapter.

## `templates/_helpers.tpl` — reusable snippets

Files starting with an underscore aren't rendered into Kubernetes objects. Instead they
hold **named templates** — reusable snippets you can include elsewhere, most commonly a
standard set of labels and a naming helper. More on these in chapter 4.

## `templates/NOTES.txt` — the post-install message

`NOTES.txt` is a template too, but its rendered output is printed to the terminal after
`helm install`. It's the perfect place to tell users how to reach the app they just
deployed.

## `charts/` — subcharts (dependencies)

The `charts/` folder holds **dependency charts**. If your app needs Redis or Postgres,
you can declare those as dependencies and Helm places them here. We'll cover
dependencies in chapter 7.

## `.helmignore`

Like `.gitignore`, this lists files to exclude when packaging the chart (`.git`,
editor files, and so on).

## Render it to see the result

You don't need a cluster to see what a chart produces. `helm template` renders
everything locally:

```bash
helm template my-demo ./demo
```

Try changing a value and re-rendering:

```bash
helm template my-demo ./demo --set replicaCount=3
```

You'll see the `replicas` field in the output change to `3`. That immediate feedback
loop — edit values, re-render, inspect — is how you'll develop every chart.

## Lint early, lint often

Helm ships a linter that catches structural mistakes:

```bash
helm lint ./demo
```

The scaffolded `demo` chart for this chapter is in
[02-anatomy-of-a-chart](https://github.com/VersusControl/devops-vn-blog/tree/main/_resource/helm-series/02-anatomy-of-a-chart).

Now that we know the layout of a chart, it's time to learn the language that powers the
templates. In the [next chapter](/helm-03-templating-basics/) we'll cover Helm
templating from the ground up.
