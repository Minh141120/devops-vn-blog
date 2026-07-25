---
layout: post
title: "Helm Templating Basics"
series: "Helm"
series_url: /helm-series/
part: 3
date: 2026-01-26
author: Quan Huynh
subtitle: "Values, built-in objects, pipelines, and functions — the core of Go templating that turns static YAML into a flexible chart."
tags: [kubernetes, helm, devops]
image: /assets/images/posts/helm-03-templating-basics/cover.svg
---

Templates are the heart of Helm. Under the hood Helm uses Go's template engine plus a
large library of helper functions. Don't worry if you've never seen Go — the syntax you
need is small and we'll build it up piece by piece.

## Template actions

Anything between double curly braces is a **template action** that Helm evaluates:

{% raw %}
```yaml
metadata:
  name: {{ .Release.Name }}
```
{% endraw %}

Everything else is copied through verbatim. So a template is just your normal YAML with
small dynamic holes punched in it.

## Reading values

The most common action is reading a value from `values.yaml` through the `.Values`
object. Given:

```yaml
# values.yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.27"
```

you reference nested keys with dots:

{% raw %}
```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  containers:
    - image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```
{% endraw %}

## Built-in objects

Besides `.Values`, Helm gives you several **built-in objects**. The ones you'll use most:

- `.Release.Name` — the release name (e.g. `my-redis`).
- `.Release.Namespace` — the target namespace.
- `.Release.Service` — always `Helm`.
- `.Chart.Name`, `.Chart.Version`, `.Chart.AppVersion` — fields from `Chart.yaml`.
- `.Values` — your merged values.
- `.Files` — access non-template files bundled in the chart.
- `.Capabilities` — what the target cluster supports (API versions, Kube version).

For example, a fully-qualified resource name often combines the release name and chart
name:

{% raw %}
```yaml
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
```
{% endraw %}

## Pipelines

Helm borrows the Unix **pipe** idea: send a value through one or more functions with
`|`. The value on the left becomes the *last* argument to the function on the right.

{% raw %}
```yaml
# uppercase the release name
name: {{ .Release.Name | upper }}

# provide a fallback if the value is empty
tag: {{ .Values.image.tag | default .Chart.AppVersion }}

# quote a string so YAML treats it as text
version: {{ .Chart.AppVersion | quote }}
```
{% endraw %}

Pipelines chain neatly:

{% raw %}
```yaml
name: {{ .Values.name | default "web" | lower | trunc 63 }}
```
{% endraw %}

## Handy functions

Helm includes the [Sprig](https://masterminds.github.io/sprig/) function library. A few
you'll use constantly:

- `default DEFAULT VALUE` — fall back when a value is missing.
- `quote` / `squote` — wrap in double / single quotes.
- `upper` / `lower` / `title` — change case.
- `trunc N` and `trimSuffix` — trim strings (useful to respect the 63-char name limit).
- `nindent N` — indent a block by N spaces, starting with a newline (essential for
  embedding blocks of YAML).
- `toYaml` — turn a values structure back into YAML.

`toYaml` with `nindent` is the classic combo for passing through a whole block, like
resource limits:

{% raw %}
```yaml
resources:
  {{- toYaml .Values.resources | nindent 2 }}
```
{% endraw %}

## Whitespace control

YAML is whitespace-sensitive, so Helm gives you dash modifiers to trim it. A `{% raw %}{{-{% endraw %}`
chomps whitespace (including the newline) *before* the action; `{% raw %}-}}{% endraw %}`
chomps *after* it.

{% raw %}
```yaml
metadata:
  labels:
    {{- if .Values.extraLabel }}
    team: platform
    {{- end }}
```
{% endraw %}

Without the leading dashes you'd get blank lines where the `if`/`end` used to be. When
your rendered YAML has odd indentation or empty lines, whitespace control is almost
always the fix — and `helm template` shows you the exact result.

## Comments

Template comments never reach the output:

{% raw %}
```yaml
{{/* This explains the next block and is stripped at render time. */}}
```
{% endraw %}

## Putting it together

Here's a small but realistic container spec using several of these ideas:

{% raw %}
```yaml
containers:
  - name: {{ .Chart.Name }}
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
    imagePullPolicy: {{ .Values.image.pullPolicy | default "IfNotPresent" }}
    ports:
      - containerPort: {{ .Values.service.port }}
```
{% endraw %}

The example `webapp` chart for this chapter is in
[03-templating-basics](https://github.com/VersusControl/devops-vn-blog/tree/main/_resource/helm-series/03-templating-basics).

We can now read values and shape them with functions. In the
[next chapter](/helm-04-flow-control-and-helpers/) we'll add logic — conditionals,
loops, and reusable named templates — which is where charts get truly powerful.
