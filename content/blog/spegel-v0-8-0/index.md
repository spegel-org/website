---
title: Announcing Spegel v0.8.0
slug: spegel-v0.8.0
date: 2026-10-16
params:
  authors:
    - name: Philip Laine
      link: https://github.com/phillebaba
---

The last minor release of Spegel was six months ago, which gave us plenty of time to add some interesting features in between patch releases and bug fixes. [For this release](https://github.com/spegel-org/spegel/releases/tag/v0.8.0) the focus has been centered around long term stability - which can affect the time in between minor releases. On the bright side, it does mean users receive a release blog, written by a human.

Refactoring was implemented in a multiple step strategy - and I would love to say that I approached it with this mindset. But looking back, it sounds much better to say I planned it in many phases other than it just “occurred” in many phases. There were [critical Containerd issues](https://github.com/spegel-org/spegel/issues/1096) and along with enabling [future work to support CRI-O](https://github.com/spegel-org/spegel/pull/1119).

*Now that this is done, I can go play with my dogs.*

![Obligatory dog picture.](./dog.jpg)

## Containerd

The Containerd component has been refactored to better handle specific edge cases for images. Specifically images that have been imported locally rather than pulled from a remote registry. This is useful for those running an air-gapped cluster where an image is loaded onto a single node and Spegel distributes the image to the rest of the cluster. Integration tests have also been added to verify this functionality in all supported Containerd versions. This does come at the cost of reducing metric granularity due to the lack of detailed information in Containerd events. Work is underway to improve some of the events in Containerd, but will take time.

With Containerd 2.4 released the minimum version has shifted. Spegel now requires Containerd 2.2 or later which are the versions currently receiving patch releases. With Containerd we are able to validate runtime configuration once again, something that has not been possible since 1.7, which will hopefully mitigate one of the most common issues when getting started with Spegel.

## Kubernetes

There has finally been more than three Kubernetes releases supporting the traffic distribution feature. Using the preferred same node traffic distribution setting on the registry service means that we can use a single mirror endpoint that prioritizes the local Spegel instance while also falling back to another node when the local instance is not an option. Previously we had to use both a pod host port along with a node port service to achieve the same functionality. With the downside of requiring two mirror endpoints, which added an extra HTTP request for images not present in the cluster. Kubernetes 1.35 is now the minimum Kubernetes version that Spegel officially supports.

A new opt-in network policy has been added to the Helm chart to strengthen the security of Spegel, by limiting libp2p communication to the namespace. This is to mitigate the chance of a bad actor joining the DHT and advertising incorrect digests for tags.

## GKE Support

Spegel now has a way to [support GKE](https://spegel.dev/docs/getting-started/#gke) without requiring Containerd to restart on a running node. This was the last major Kubernetes provider that Spegel could not run on. There is an [open issue](https://issuetracker.google.com/issues/562778580) to improve the support in GKE. Please make your voice heard if this affects you as it helps Google prioritize the work that is important for end users.

## Observability

I had to rework some of the metrics to be able to implement support for imported images in Containerd. The following metrics.

```
spegel_advertised_images
spegel_advertised_image_tags
spegel_advertised_image_digests
```

Has been replaced by the single metric `spegel_advertised_keys`. The metric serves the same purpose, showing that the router is working and how many keys have been advertised. Due to limitations of how Containerd works we are no longer able to include a registry label in the metric.

The change in metrics means that the Grafana dashboard will not be compatible with the new version metrics. This is why the Grafana dashboard has been removed from the Helm chart. Community members have varying needs for Grafana dashboards, and maintaining one “standard” is quite difficult. For those relying on the dashboard the easiest way forward is to copy the dashboard from the last minor release and maintain it separately.

## The Future

Spegel is a project I have worked on for the last five years. It has brought me a lot of joy and frustration. This release brings stability so that future Spegel releases will be focused on CVE fixes and dependency upgrades. I will be shifting my focus on [Kvick’s Spegel Enterprise Edition (SEE)](https://docs.kvick.dev/spegel-enterprise/overview/) to support the enterprise use cases of Spegel.
