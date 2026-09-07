---
title: Overview
weight: 1
aliases:
  - /docs/
---

Kubernetes does a great job at distributing workloads on multiple nodes. Allowing node failures to occur without affecting uptime. A critical component for this to work is that each node has to pull the workload images before the container can start.

The images may be pulled from geographically close registries within the cloud provider, public registries, or self-hosted registries. This process has a flaw in that each node has to make this round trip separately. Even if the data they download is identical.

## What is Spegel?

<span style="font-style: italic; color: var(--hx-color-gray-500)">Spegel enables nodes to share images with each other.</span>

Spegel is a stateless cluster local OCI registry mirror that enables nodes to bypass upstream registries by pulling from other nodes in the cluster. Any image already pulled by a node will be available for any other node in the cluster to pull.

## How does Spegel work?

Spegel runs a read only OCI registry on every host in a cluster. It integrates with Containerd to access images that have been pulled and cached by Containerd on the host. That way no additional storage space is needed.

{{< inline-vector-figure "./architecture.svg" >}}

When an image is pulled the request is first redirected to the local Spegel instance. Each instance is a member of a DHT, enabling quick lookups of images. If another instance has the required image the request is forwarded and served from the Containerd image cache. If the image cannot be found in any instance it will fallback to pulling from the upstream registry.

## Who is Spegel for?

Spegel is for any platform engineer who wants to improve the experience for their end users.

* Improve container startup time by reducing image pull time.
* Reduce uptime dependencies on external registries.
* Minimize NAT egress costs related to image distribution.

If this sound like you, jump to the getting started guide and get Spegel running in 5 minutes.

