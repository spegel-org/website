---
title: Announcing Spegel v0.1.0
slug: spegel-v0.1.0
date: 2025-03-22
params:
  authors:
    - name: Philip Laine
      link: https://github.com/phillebaba
---

Spegel’s journey began with its first commit in February 2023, sparked by a curiosity about OCI and Containerd, along with a challenge to explore alternative image distribution methods. In the two years since, it has evolved into a valuable tool for everyone from homelab enthusiasts to multinational corporations. Seeing the GitHub stars climb has been rewarding, a clear indicator that adoption is growing and people recognize its value.

![Star History](/blog/spegel-v0-1-0/star-history.png)

Since Spegel’s initial release, breaking changes have been avoided, and for the most part, the project has evolved smoothly. However, the time has come to address past missteps. The [v0.1.0 release](https://github.com/spegel-org/spegel/releases/tag/v0.1.0) marks an important milestone, enabling necessary changes while also committing to proper versioning and stability moving forward. It’s important to review the changelog to understand if these breaking changes might impact you. From here on, future releases will follow semantic versioning, with new features introduced through minor version increases. While this release could have come sooner, uncertainty in Spegel’s implementation made it challenging. But, as they say, better late than never.

Before diving into the new features and changes, a quick request. If your company is using Spegel, adding your organization to the adopters page would be extremely valuable. One of the biggest challenges in open source is understanding who the end users are. If you find value in Spegel and want to support its continued development, consider sponsoring the project. Your support helps ensure the future of Spegel and its ongoing improvements.

<iframe src="https://github.com/sponsors/phillebaba/card" title="Sponsor phillebaba" style="border: 0; margin-top: 20px; width: 100%; height: 120px;"></iframe>

We’re building a stronger community of Spegel users, and we’d love for you to be part of it. A Slack channel has been created within the Kubernetes Slack organization, and we hold community meetings every Tuesday at 17:00 CET. To learn how to join us, check out the [community documentation](https://spegel.dev/project/community/).

## Mirror all registries

One of the most significant breaking changes in v0.1.0 is that Spegel will now configure all registries to be mirrored by default. This shift has been a long time coming and is the primary driver behind this release. Initially, only a limited set of default registries were configured to minimize potential negative impacts if something went wrong. Over the past two years, however, confidence in Spegel has grown, and most users now prefer to mirror all images from all registries instead of just a select few.

Along with this change, several configuration values have been renamed for clarity. The setting `spegel.appendMirrors` is now `spegel.prependMirrors` to more accurately describe its function. Similarly, `spegel.registries` has been renamed to `spegel.mirroredRegistries`, and `spegel.additionalMirrorRegistries` has been renamed to `spegel.additionalMirrorTargets`. These changes aim to reduce confusion and more clearly distinguish between the registries to be mirrored and the target mirrors where requests will be sent.

## Debug Web UI

A challenge with Spegel has been gaining insights into its operation. New users often struggle to determine whether Spegel is working correctly. This is a significant issue. Until now, users have had to rely on container logs and HTTP status codes to figure out if an image was found and pulled successfully. To simplify this process, a new debug web UI has been added. It is disabled by default but can be easily enabled by setting `spegel.debugWebEnabled: true`. To access it, simply port forward to a Spegel pod on its metrics port and browse to `http://localhost:9090/debug/web`.

![Debug Web](/blog/spegel-v0-1-0/debug-web.png)

While the features are still limited, you can already initiate an image pull from other Spegel nodes to check that images are found and pulled correctly. We welcome any feedback. If there is additional information that would help with troubleshooting, please consider creating an issue.

## Memory requests and limits

The Helm chart now includes default memory requests and limits to address feedback about excessive memory usage by Spegel. Linux file reading relies on the VFS cache, which keeps file chunks in memory for efficiency. In a containerized environment, VFS cache usage contributes to the container’s overall memory consumption. Since the VFS cache is only cleared under memory pressure, Spegel may appear to consume increasing amounts of memory until it reaches the available memory on the node. While some users have manually set memory limits to control this, the majority likely have not. To prevent confusion about excessive memory usage, the Helm chart now sets a default memory limit of 128Mi, ensuring the cache is cleared when necessary. This change helps support the majority of users and avoids future issues.

## New bootstrapper

This feature was originally part of v0.0.29 but was intended for the v0.1.0 release. Due to the delay of v0.1.0, it was made available earlier. The new bootstrapper now uses a headless DNS service to find peers within the cluster. This change addresses the fact that Kubernetes leader election was never designed to be used by a DaemonSet in large Kubernetes clusters. Users running clusters with 500+ nodes were facing performance issues, as all nodes were communicating with the API server. The old Kubernetes bootstrap method has now been fully removed, eliminating any Go Kubernetes dependencies. This new method is more lightweight and should significantly improve startup times.

## Future

There’s a lot of exciting work ahead to make Spegel faster and more efficient. For a glimpse of the major features being prioritized, check out the [roadmap](https://github.com/orgs/spegel-org/projects/3). Looking ahead, a v1.0.0 release of Spegel is in the plans, though it’s still some time away. This release will likely align with the wider adoption of Containerd v2, which would allow Spegel to drop support for Containerd v1.7 and earlier versions. The timing will depend on how quickly cloud providers and end users adopt Containerd v2.

Spegel development will continue to grow and evolve, with exciting new features ahead. Thanks to your support, we’re building something truly special. Here’s to another year of keeping things fast and local, without the need to pull from upstream registries!
