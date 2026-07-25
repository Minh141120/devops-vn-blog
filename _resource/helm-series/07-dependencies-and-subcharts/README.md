# Chapter 7 — Dependencies and Subcharts

This umbrella chart adds a **Redis** cache to Book Info via the CloudPirates Redis
OCI chart. It shows the dependency wiring only — combine it with the Book Info
templates from [chapter 5](../05-charting-book-info) for a complete chart.

Fetch the dependency (writes `Chart.lock` and populates `charts/`), then install:

```bash
helm dependency update ./book-info
helm install book-info ./book-info

# turn Redis off for dev
helm install book-info-dev ./book-info -f ./book-info/values-dev.yaml
```
