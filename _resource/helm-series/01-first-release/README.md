# Chapter 1 — Your First Helm Release

Install the CloudPirates Redis OCI chart directly by its URL (no `helm repo add` needed):

```bash
helm install my-redis oci://registry-1.docker.io/cloudpirates/redis --version 0.33.0
```

Install with the example values file in this folder:

```bash
helm install my-redis oci://registry-1.docker.io/cloudpirates/redis --version 0.33.0 -f my-values.yaml
```

Everyday commands:

```bash
helm list
helm get values my-redis
helm upgrade my-redis oci://registry-1.docker.io/cloudpirates/redis --version 0.33.0 --set replicaCount=3
helm rollback my-redis 1
helm uninstall my-redis
```
