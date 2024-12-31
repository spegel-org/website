---
title: Getting Started
weight: 1
---

Before installing Spegel check the [compatibility guide](./docs/COMPATIBILITY.md) to make sure that it will work with your specific Kubernetes flavor. If everything checks out, the easiest method to deploy Spegel is with Helm.

```shell
helm upgrade --create-namespace --namespace spegel --install --version v0.0.27 spegel oci://ghcr.io/spegel-org/helm-charts/spegel
```

Refer to the [Helm Chart](./charts/spegel) for detailed configuration documentation.


