---
title: Local Cluster
weight: 3
---

Spegel is designed for production Kubernetes clusters where images are shared between nodes using a stateless, best effort OCI registry mirror. It is optimized for reducing registry traffic, improving pull performance, and increasing resilience across many nodes.

For local Kubernetes clusters, we recommend using [Hall](https://hall.kvick.dev/) instead. Hall is built specifically for local Kubernetes development and is powered by Spegel, importing Spegel as a library to provide image distribution capabilities tailored for local environments.

## Using Hall

After installing Hall, start it alongside your local cluster:

```sh
hall start
```

Build your application image as usual:

```sh
docker build -t my-app:latest .
```

You can then reference the image directly from your Kubernetes manifests:

```yaml
spec:
  containers:
    - name: app
      image: my-app:latest
```

Whenever you rebuild the image, Hall automatically makes the updated image available to your local cluster. No registry, `kind load`, or image import commands are required.
