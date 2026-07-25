---
layout: post
title: "Packaging, Registries, and Best Practices"
series: "Helm"
series_url: /helm-series/
part: 9
date: 2026-03-09
author: Quan Huynh
subtitle: "Package a chart, publish it to an OCI registry, and finish with the conventions that keep charts maintainable — the habits of a Helm pro."
tags: [kubernetes, helm, devops]
image: /assets/images/posts/helm-09-packaging-and-best-practices/cover.svg
---

We've built, deployed, and operated the Book Info chart. In this final chapter we
**share** it — packaging it into an archive and pushing it to a registry — then close the
series with the best practices that separate a throwaway chart from a maintainable one.

## Packaging a chart

Bundle a chart into a versioned `.tgz` archive:

```bash
helm package ./book-info
```

```
Successfully packaged chart and saved it to: book-info-0.1.0.tgz
```

The filename comes from `name` and `version` in `Chart.yaml` — which is exactly why you
bump `version` on every change. You can install straight from the archive:

```bash
helm install book-info ./book-info-0.1.0.tgz
```

## Publishing to an OCI registry

Helm 4 treats **OCI registries as first-class** chart storage — the same registries that
hold your container images (Docker Hub, GHCR, ECR, Harbor, …). No separate chart server
to run.

Log in, then push the packaged chart:

```bash
helm registry login registry.example.com
helm push book-info-0.1.0.tgz oci://registry.example.com/charts
```

Pull or install from the registry by its `oci://` URL:

```bash
helm install book-info oci://registry.example.com/charts/book-info --version 0.1.0
```

You can also declare an OCI chart as a dependency, just like an HTTP repo:

```yaml
dependencies:
  - name: redis
    version: "0.33.0"
    repository: oci://registry-1.docker.io/cloudpirates
```

## The classic HTTP repository (still supported)

If you prefer a traditional repo, `helm package` plus an index still works. Generate an
`index.yaml` over a folder of packaged charts and serve it over HTTP (GitHub Pages is a
popular free host):

```bash
helm repo index ./charts --url https://my-org.github.io/charts
```

Consumers then `helm repo add` your URL and install charts from it — the classic
HTTP-repository workflow, an alternative to the OCI approach we've used elsewhere.

## Best practices

A checklist distilled from the whole series.

**Structure & naming**
- Keep charts **small and focused**; prefer several charts over one giant umbrella.
- Use the standard `app.kubernetes.io/*` labels via a `_helpers.tpl` labels template.
- Respect the 63-character name limit — `{% raw %}{{ ... | trunc 63 | trimSuffix "-" }}{% endraw %}`.

**Values**
- Provide sensible **defaults** in `values.yaml`; keep per-environment overlays thin.
- Document every value with a comment, and ship a `values.schema.json` to validate input
  where it matters.
- Model repeated things (like our services) as **maps**, so overlays can patch one field
  without redeclaring the list.

**Templating**
- Prefer `include` over `template` so you can pipe through `nindent`.
- Use `required` and `fail` to turn misconfiguration into clear errors early.
- Never hard-code namespaces, image tags, or replica counts — surface them as values.

**Safety & workflow**
- `helm lint`, `helm template`, and `--dry-run` **before** every apply.
- Upgrade with `--atomic --wait` so failures roll back automatically.
- Commit `Chart.lock` so dependency versions are reproducible.
- Bump `Chart.yaml` `version` on every change; treat charts like versioned software.

**Secrets**
- Never put plaintext secrets in `values.yaml`. Use sealed secrets, an external secrets
  operator, or a secrets plugin — the same rule we followed in the ArgoCD series.

## Where to go next

You now have the full toolkit: you can consume community charts, author your own from
scratch, template them cleanly, manage values across environments, compose dependencies,
operate upgrades and rollbacks, and publish to a registry.

The natural next step is **GitOps** — letting a controller apply your charts from Git
automatically. If you haven't yet, head over to the
[ArgoCD series](/argocd-series/), whose *Working with Helm* chapter deploys charts
exactly like the Book Info one you built here.

The packaging files (a `values.schema.json` plus the commands) are in
[09-packaging-and-best-practices](https://github.com/VersusControl/devops-vn-blog/tree/main/_resource/helm-series/09-packaging-and-best-practices).

That's a wrap on the Helm series. Happy charting!
