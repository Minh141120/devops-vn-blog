---
layout: post
title: "Flow Control and Named Templates"
series: "Helm"
series_url: /helm-series/
part: 4
date: 2026-02-02
author: Quan Huynh
subtitle: "Conditionals, loops with range, scope with with, and reusable snippets with define/include — the logic that makes charts DRY."
tags: [kubernetes, helm, devops]
image: /assets/images/posts/helm-04-flow-control-and-helpers/cover.svg
---

In the last chapter we learned to read and shape values. Now we add **logic**:
generating resources conditionally, looping to avoid repetition, and factoring out
reusable snippets. These are the tools that let one small chart deploy a whole
application.

## Conditionals: if / else

Render a block only when a condition is true:

{% raw %}
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}
{{- end }}
```
{% endraw %}

You can add `else` and `else if` too:

{% raw %}
```yaml
type: {{ if .Values.service.external }}LoadBalancer{{ else }}ClusterIP{{ end }}
```
{% endraw %}

Helm treats these as *false*: `false`, `0`, an empty string, an empty list, an empty
map, and `nil`. Everything else is true.

## Loops: range

`range` iterates over a list or a map — perfect for generating N similar resources. Over
a **list**:

{% raw %}
```yaml
env:
{{- range .Values.extraEnv }}
  - name: {{ .name }}
    value: {{ .value | quote }}
{{- end }}
```
{% endraw %}

Inside the loop, `.` (the dot) is rebound to the current item. Over a **map** you get
both key and value:

{% raw %}
```yaml
{{- range $key, $value := .Values.annotations }}
{{ $key }}: {{ $value | quote }}
{{- end }}
```
{% endraw %}

## The dot and the `$` root

This trips up every beginner, so it's worth stating clearly: inside a `range` (or
`with`), the dot `.` changes to point at the current item. If you still need the
top-level context — `.Values`, `.Release`, `.Chart` — use the **root** object `$`,
which always points at the very top:

{% raw %}
```yaml
{{- range $name, $svc := .Values.services }}
  image: "{{ $.Values.image.registry }}/{{ $svc.repository }}"
{{- end }}
```
{% endraw %}

Here `$svc` is the current service, but `$.Values.image` reaches back to the global
image settings. We'll rely on exactly this pattern when we template the Book Info app.

## Scope: with

`with` narrows the dot to a sub-object, saving repetition:

{% raw %}
```yaml
{{- with .Values.resources }}
resources:
  requests:
    cpu: {{ .requests.cpu }}
    memory: {{ .requests.memory }}
{{- end }}
```
{% endraw %}

Inside the `with`, `.` is `.Values.resources`. Bonus: the block is skipped entirely if
`resources` is empty — a neat way to make optional sections disappear.

## Named templates: define and include

Repetition across files (labels, selectors, names) is best factored into **named
templates** in `_helpers.tpl`. Define one with `define`:

{% raw %}
```yaml
{{- define "book-info.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}
```
{% endraw %}

Then pull it in wherever you need it with `include`, piping through `nindent` to get the
indentation right:

{% raw %}
```yaml
metadata:
  labels:
    {{- include "book-info.labels" . | nindent 4 }}
```
{% endraw %}

> **Why `include` and not `template`?** Helm has a `template` action too, but it can't be
> piped into functions like `nindent`. `include` returns the rendered text as a string,
> so it composes with pipelines. **Always prefer `include`.**

Notice we pass `.` as the second argument — that hands the current context to the named
template so it can read `.Chart`, `.Release`, and so on. Inside a `range`, you'd pass
`$` instead to give it the root context.

## fail and required — guard rails

You can stop a render with a clear error when a required value is missing:

{% raw %}
```yaml
image: {{ required "image.repository is required!" .Values.image.repository }}
```
{% endraw %}

This is far friendlier than shipping a broken manifest to the cluster.

## A complete helper file

Putting it together, a typical `_helpers.tpl` provides a name helper and a labels
helper that every resource reuses:

{% raw %}
```yaml
{{- define "book-info.name" -}}
{{ .Chart.Name }}
{{- end -}}

{{- define "book-info.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version }}
{{- end -}}
```
{% endraw %}

The example `webapp` chart (with `_helpers.tpl` and flow control) is in
[04-flow-control-and-helpers](https://github.com/VersusControl/devops-vn-blog/tree/main/_resource/helm-series/04-flow-control-and-helpers).

We now have every building block we need. In the
[next chapter](/helm-05-charting-book-info/) we put it all to work and turn the Book
Info microservices into a real, reusable chart.
