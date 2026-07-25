---
layout: post
title: "Charting the Book Info App"
series: "Helm"
series_url: /helm-series/
part: 5
date: 2026-02-09
author: Quan Huynh
subtitle: "Turn the four Book Info microservices into one clean, DRY Helm chart driven entirely by values — then install it with a single command."
tags: [kubernetes, helm, devops]
image: /assets/images/posts/helm-05-charting-book-info/cover.svg
---

Time to build something real. The **Book Info** app has four services — a `productpage`
frontend plus `details`, `ratings`, and `reviews` backends. Each needs a Deployment, a
Service, and a ServiceAccount. That's twelve Kubernetes objects. Instead of twelve
files, we'll write **one** templated set driven by values.

The finished chart lives at
[05-charting-book-info/book-info](https://github.com/VersusControl/devops-vn-blog/tree/main/_resource/helm-series/05-charting-book-info/book-info)
and uses the current Book Info images (`1.20.3`).

## The plan

All four services share the same shape — only the name, image, replica count, and
service name differ. That's a textbook case for a **`range` over a map of services**.
We describe the services in `values.yaml` and let one template generate all of them.

```
book-info/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-prod.yaml
└── templates/
    ├── _helpers.tpl
    ├── serviceaccount.yaml
    ├── service.yaml
    ├── deployment.yaml
    └── NOTES.txt
```

## Chart.yaml

```yaml
apiVersion: v2
name: book-info
description: A Helm chart for the Book Info microservices sample (Helm 4)
type: application
version: 0.1.0
appVersion: "1.20.3"
```

## values.yaml — describe the services as data

The key idea: the *configuration is data*. Shared image settings live at the top, then a
map where each entry is one service.

```yaml
image:
  registry: docker.io/istio
  tag: "1.20.3"
  pullPolicy: IfNotPresent

services:
  productpage:
    repository: examples-bookinfo-productpage-v1
    serviceName: products
    account: bookinfo-productpage
    replicas: 1
    port: 9080
  details:
    repository: examples-bookinfo-details-v1
    serviceName: details
    account: bookinfo-details
    replicas: 1
    port: 9080
  ratings:
    repository: examples-bookinfo-ratings-v1
    serviceName: ratings
    account: bookinfo-ratings
    replicas: 1
    port: 9080
  reviews:
    repository: examples-bookinfo-reviews-v3
    serviceName: reviews
    account: bookinfo-reviews
    replicas: 1
    port: 9080
```

> Book Info ships three versions of `reviews`. To keep the chart approachable we deploy
> one version at a time (v3 by default) and switch it per environment with a values
> override — you'll see that in the next chapter.

## _helpers.tpl — shared labels

```yaml
{% raw %}{{- define "book-info.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version }}
{{- end -}}{% endraw %}
```

## serviceaccount.yaml — one per service

We `range` over the services map. Remember from chapter 4: inside the loop `.` becomes
the current item, so we use `$` to reach the root context (needed by the labels helper).

{% raw %}
```yaml
{{- range $name, $svc := .Values.services }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ $svc.account }}
  labels:
    {{- include "book-info.labels" $ | nindent 4 }}
    account: {{ $name }}
---
{{- end }}
```
{% endraw %}

## service.yaml — one Service per app

The `serviceName` matters: the `productpage` frontend talks to backends by their service
names (`details`, `reviews`, `ratings`), so those must be exact.

{% raw %}
```yaml
{{- range $name, $svc := .Values.services }}
apiVersion: v1
kind: Service
metadata:
  name: {{ $svc.serviceName }}
  labels:
    {{- include "book-info.labels" $ | nindent 4 }}
    app: {{ $name }}
spec:
  ports:
    - port: {{ $svc.port }}
      name: http
  selector:
    app: {{ $name }}
---
{{- end }}
```
{% endraw %}

## deployment.yaml — the workhorse

One template, four Deployments. Note how the image is assembled from the shared
`$.Values.image` settings and the per-service `repository`:

{% raw %}
```yaml
{{- range $name, $svc := .Values.services }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ $name }}
  labels:
    {{- include "book-info.labels" $ | nindent 4 }}
    app: {{ $name }}
spec:
  replicas: {{ $svc.replicas | default 1 }}
  selector:
    matchLabels:
      app: {{ $name }}
  template:
    metadata:
      labels:
        {{- include "book-info.labels" $ | nindent 8 }}
        app: {{ $name }}
    spec:
      serviceAccountName: {{ $svc.account }}
      containers:
        - name: {{ $name }}
          image: "{{ $.Values.image.registry }}/{{ $svc.repository }}:{{ $.Values.image.tag }}"
          imagePullPolicy: {{ $.Values.image.pullPolicy }}
          ports:
            - containerPort: {{ $svc.port }}
---
{{- end }}
```
{% endraw %}

## NOTES.txt — tell the user what to do next

{% raw %}
```
Book Info deployed as release "{{ .Release.Name }}" in namespace "{{ .Release.Namespace }}".

Open the product page locally:
  kubectl port-forward svc/{{ (index .Values.services "productpage").serviceName }} 9080:9080
Then visit http://localhost:9080/productpage
```
{% endraw %}

## Render and check

Before installing, always render locally and eyeball the output:

```bash
helm template book-info ./book-info | grep 'image:'
```

```
image: "docker.io/istio/examples-bookinfo-details-v1:1.20.3"
image: "docker.io/istio/examples-bookinfo-productpage-v1:1.20.3"
image: "docker.io/istio/examples-bookinfo-ratings-v1:1.20.3"
image: "docker.io/istio/examples-bookinfo-reviews-v3:1.20.3"
```

Lint it too:

```bash
helm lint ./book-info
```

## Install it

```bash
helm install book-info ./book-info --namespace book-info --create-namespace
```

Watch the pods come up, then open the app:

```bash
kubectl get pods -n book-info
kubectl port-forward -n book-info svc/products 9080:9080
```

Visit `http://localhost:9080/productpage` and you'll see Book Info running — all four
services deployed from a single, DRY chart.

The complete Book Info chart is in
[05-charting-book-info](https://github.com/VersusControl/devops-vn-blog/tree/main/_resource/helm-series/05-charting-book-info).

Adding a fifth service would now mean adding **four lines to `values.yaml`** — no new
templates. That's the payoff of data-driven charts. In the
[next chapter](/helm-06-values-and-environments/) we'll use that same design to deploy
dev and prod from one chart.
