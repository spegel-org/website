---
title: Announcing Spegel v0.3.0
slug: spegel-v0.3.0
date: 2025-06-06
params:
  authors:
    - name: Philip Laine
      link: https://github.com/phillebaba
---

[Containerd v2.1.0](https://github.com/containerd/containerd/releases/tag/v2.1.0) unlocks a major piece of the puzzle for Spegel. This release removes key limitations that previously held back development. A big part of the work in v0.3.0 has been making sure everything plays well with the new version. That meant refactoring internals and updating the end-to-end tests to run against it. It will take some time before this version makes its way into most cloud providers, but once it does, Spegel will be ready.

## Content Events

Containerd now supports content events, introduced in [#11006](https://github.com/containerd/containerd/pull/11006). What this means for you as the end user is faster advertisement of layers. Previously, Spegel wouldn’t be notified about new layers until the entire image had been pulled. With large images, the delay between one layer being successfully pulled and the next could be several minutes. This meant that even if the layer was already present on the node, we wouldn’t advertise it. The issue became especially noticeable when multiple pods using the same image were started at the same time.

Large parts of the event logic have now been refactored to support both image and layer events. As a result, layers are advertised as soon as they are available. While this may not improve performance across the board, it can make a noticeable difference in specific use cases. Once the benchmarking tool is further developed, we’ll be able to measure more precisely what impact early layer advertisement has on overall Spegel performance.

## Dial Timeout

Containerd now supports dial timeouts for mirrors, introduced in [#11106](https://github.com/containerd/containerd/pull/11106). Previously, if the local Spegel instance failed to start for any reason, Containerd would wait indefinitely for the TCP connection to be established. If Spegel never came up, Containerd would never move on to another mirror or fall back to the upstream registry. This could effectively block the node from pulling any images, which is obviously not great. With the new dial timeout parameter, we can set a limit to avoid this situation. Spegel now includes this parameter when writing mirror configurations. Since the traffic is local, connection times are expected to be low. After some testing, one second was found to be more than enough to establish a TCP connection within the cluster. While real-world cases have been rare, this is still a valuable stability improvement.

## Containerd Future Work

Containerd includes two new features that will impact Spegel in future releases. The first is [OCI volume support](https://github.com/containerd/containerd/pull/10579), which allows containers to mount other OCI artifacts as volumes. These are pulled using the same mechanism as regular container images, so they should work seamlessly with Spegel. The documentation will be updated once the end-to-end tests have been extended to cover this.

The second is multipart fetching, which was merged after a year in development. This feature splits layer pulling into smaller chunks and downloads them in parallel, improving pull speed. Although currently opt-in, [multipart fetching](https://github.com/containerd/containerd/pull/10177) could bring significant performance gains when combined with Spegel. Exploring its impact will be an important next step.
