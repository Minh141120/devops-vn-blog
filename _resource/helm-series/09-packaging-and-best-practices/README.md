# Chapter 9 — Packaging and Best Practices

Package the chart into a versioned archive and push it to an OCI registry:

```bash
# package ./book-info (from chapter 5) into book-info-0.1.0.tgz
helm package ../05-charting-book-info/book-info

# publish to an OCI registry
helm registry login registry.example.com
helm push book-info-0.1.0.tgz oci://registry.example.com/charts

# install from the registry
helm install book-info oci://registry.example.com/charts/book-info --version 0.1.0
```

`values.schema.json` in this folder shows how to validate user-provided values: drop it
at the root of a chart and Helm rejects installs whose values don't match the schema.
