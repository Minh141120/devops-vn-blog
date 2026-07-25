# Helm Series — Charts & Manifests

Ready-to-use charts and manifests that accompany the
[Helm series](https://devopsvn.tech/helm-series/) on DevOps VN.

Each folder maps to a chapter of the series:

| Chapter | Folder | What's inside |
|--------|--------|---------------|
| 1 | `01-first-release` | Example values file for the CloudPirates Redis OCI chart |
| 2 | `02-anatomy-of-a-chart` | A scaffolded `demo/` chart (Chart.yaml, values, templates) |
| 3 | `03-templating-basics` | A `webapp/` chart showing values, built-ins, and pipelines |
| 4 | `04-flow-control-and-helpers` | A `webapp/` chart with `_helpers.tpl`, `if`/`range`/`with` |
| 5 | `05-charting-book-info` | The full **Book Info** chart (`book-info/`) |
| 6 | `05-charting-book-info` | Same chart — `values-dev.yaml` / `values-prod.yaml` overlays |
| 7 | `07-dependencies-and-subcharts` | Umbrella chart with a Redis (CloudPirates) dependency |
| 8 | `08-upgrades-and-rollbacks` | Chart hooks — a pre-upgrade Job and a `helm test` Pod |
| 9 | `09-packaging-and-best-practices` | A `values.schema.json` and packaging commands |

> Redis examples use the CloudPirates Redis OCI chart
> (`oci://registry-1.docker.io/cloudpirates/redis`). The Book Info chart uses the Istio
> Book Info images (`docker.io/istio/examples-bookinfo-*:1.20.3`).

## Book Info quick start

```bash
cd 05-charting-book-info

# render & validate
helm lint ./book-info
helm template book-info ./book-info

# install
helm install book-info ./book-info --namespace book-info --create-namespace

# per-environment
helm install book-info-dev  ./book-info -f ./book-info/values-dev.yaml  -n book-info-dev  --create-namespace
helm install book-info-prod ./book-info -f ./book-info/values-prod.yaml -n book-info-prod --create-namespace

# open the app
kubectl port-forward -n book-info svc/products 9080:9080
# then visit http://localhost:9080/productpage
```
