---
layout: post
title: "Upgrades, Rollbacks, and Hooks"
series: "Helm"
series_url: /helm-series/
part: 8
date: 2026-03-02
author: Quan Huynh
subtitle: "The day-2 lifecycle — upgrade safely, roll back instantly, inspect release history, and run tasks at the right moment with chart hooks."
tags: [kubernetes, helm, devops]
image: /assets/images/posts/helm-08-upgrades-and-rollbacks/cover.svg
---

Installing an app is the easy part; living with it is where Helm really earns its keep.
This chapter covers the operational lifecycle: upgrading, rolling back, reading history,
and hooking into key moments of a release.

## Upgrading a release

An upgrade re-renders the chart with new values or a new chart version and applies the
changes:

```bash
helm upgrade book-info-prod ./book-info \
  -f ./book-info/values-prod.yaml
```

Each successful `install`/`upgrade` creates a new **revision**. A few flags make
upgrades safer:

- `--atomic` — if the upgrade fails, automatically roll back to the last good revision.
- `--wait` — wait until the new resources report ready before declaring success.
- `--timeout 5m` — how long to wait before giving up.
- `--install` — install the release if it doesn't exist yet (great for CI: "upgrade or
  install").

A production-friendly command combines them:

```bash
helm upgrade --install book-info-prod ./book-info \
  -f ./book-info/values-prod.yaml \
  --atomic --wait --timeout 5m
```

## Reading the history

Helm keeps every revision:

```bash
helm history book-info-prod
```

```
REVISION  UPDATED   STATUS      CHART            APP VERSION  DESCRIPTION
1         ...       superseded  book-info-0.1.0  1.20.3       Install complete
2         ...       superseded  book-info-0.2.0  1.20.3       Upgrade complete
3         ...       deployed    book-info-0.2.0  1.20.3       Upgrade complete
```

## Rolling back

Made a bad change? Roll back to any previous revision in one command:

```bash
helm rollback book-info-prod 2
```

Helm re-applies the manifests from revision 2 and records the rollback as a *new*
revision (so your history stays linear and auditable). With `--atomic` on your upgrades,
most bad deploys undo themselves automatically — but a manual `rollback` is always
there.

## Dry runs and diffs

Before any upgrade, preview it. A server-side dry run validates against the cluster:

```bash
helm upgrade book-info-prod ./book-info -f ./book-info/values-prod.yaml --dry-run
```

Or show a precise diff of what would change with the diff plugin from chapter 6:

```bash
helm diff upgrade book-info-prod ./book-info -f ./book-info/values-prod.yaml
```

## Hooks: run tasks at lifecycle moments

**Hooks** let a chart run Kubernetes objects (usually Jobs) at specific points in a
release's life — for example, a database migration before an upgrade. You mark a
resource as a hook with an annotation:

{% raw %}
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-migrate
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: my-migrations:1.0.0
          command: ["python", "manage.py", "migrate"]
```
{% endraw %}

The common hook points:

- `pre-install` / `post-install`
- `pre-upgrade` / `post-upgrade`
- `pre-delete` / `post-delete`
- `pre-rollback` / `post-rollback`
- `test` — run only by `helm test`

**hook-weight** orders multiple hooks (lower runs first); **hook-delete-policy** controls
when the hook's objects are cleaned up.

## Testing a release

The `test` hook is a lightweight smoke test. Put a Job or Pod under
`templates/tests/` annotated with `"helm.sh/hook": test` — for example, one that curls
the product page:

```bash
helm test book-info-prod
```

Helm runs the test resources and reports pass/fail, so you can verify a deploy actually
works end-to-end.

## Uninstalling — and keeping history

```bash
helm uninstall book-info-prod
```

By default this removes everything. Add `--keep-history` if you want the revision record
to survive (so you could `helm rollback` it back later).

The hook manifests for this chapter are in
[08-upgrades-and-rollbacks](https://github.com/VersusControl/devops-vn-blog/tree/main/_resource/helm-series/08-upgrades-and-rollbacks).

You can now operate a release with confidence. In the final chapter,
[Packaging and Best Practices](/helm-09-packaging-and-best-practices/), we'll package the
chart, push it to an OCI registry, and wrap up with the habits that keep charts healthy.
