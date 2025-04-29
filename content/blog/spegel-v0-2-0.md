---
title: Announcing Spegel v0.2.0
slug: spegel-v0.2.0
date: 2025-04-29
params:
  authors:
    - name: Philip Laine
      link: https://github.com/phillebaba
---

The focus of this release has been on cleaning up the codebase, increasing test coverage, and simplifying future extensions. A few minor, non-breaking bugs introduced in v0.1.0 have also been fixed. Beyond that, the two main improvements that matter to end users are: a new cleanup mechanism added to the Helm chart, and the introduction of prebuilt binary releases for Spegel.

## Cleanup of Mirror Configuration

The lack of proper cleanup has been a major pain point for users testing or benchmarking Spegel. Previously, Spegel would write the containerd mirror configuration to the host, but would not remove it when the Helm chart was uninstalled. This left behind mirror settings pointing to a registry that no longer existed. The result varied, but typically added unwanted latency as containerd attempted to reach a mirror that would never respond. In clusters that regularly rotate nodes, this wasn’t an issue since new nodes wouldn’t have the stale configuration. Still, this was bad practice on Spegel’s part.

The challenge was finding a solution that works with Helm’s uninstall hooks in combination with daemonsets. Helm only natively supports pod and job resources if you want it to wait for completion during uninstall. Spegel needed a way to run cleanup logic on every node and ensure it completed before uninstalling.

One option was to create a job that would in turn launch cleanup jobs for each node and wait for them to complete. But that approach is overly complex and can leave behind resources in failure scenarios.

Instead, Spegel now takes advantage of a quirk in Helm: it allows a pod and a daemonset to be created as part of the uninstall hook. The daemonset removes the mirror configuration from each host and then starts an HTTP server to respond to health probes. The accompanying pod runs a simple prober that checks all the daemonset pods. Once all respond successfully, we assume cleanup is complete. This setup allows Helm to manage all uninstall-related resources and requires no additional Kubernetes expertise from users.

This cleanup feature is enabled by default in the latest version of the Helm chart.

## Binary Releases

Some users have expressed interest in running Spegel directly on the host, outside of Kubernetes. To support this use case, the release process now includes standalone binaries. This required some refactoring of the build pipeline. Spegel is now built outside of Docker, and the Docker image build step simply copies the compiled binary into the image.

Host-level support is still experimental. The main challenge is finding the right solution for the bootstrap process. Currently, the best option is to use the static bootstrapper, though it has some limitations. It depends on specific peers being available. This can lead to issues if those peers are down.

Work will continue on improving the experience for running Spegel on the host, and documentation will follow once the current limitations are addressed.

## Refactoring the Registry Proxy

There have been two major changes to the registry mirror implementation.

First, the use of the standard library HTTP proxy has been completely removed. It never fully aligned with Spegel’s mirroring logic and was unnecessarily bulky. Spegel does not need all the features of a general-purpose proxy. It handles a narrow and well-defined set of proxying behaviors. Since Spegel is aware of the request types and supported paths, it can make assumptions a generic proxy cannot. The proxy has been replaced with a lightweight HTTP client and some custom logic instead. This also prepares the codebase for future features like the referrers API, which will require an HTTP client anyway.

The second change is the removal of request source detection. Before v0.2.0, if Spegel received a request for content that had not yet been mirrored, it would inspect the client IP. If the request came from another Spegel instance, it could allow proxying the traffic to itself if selected during lookup. If the request was local, it would avoid routing to itself and only consider other nodes. This logic was introduced early in Spegel’s development and has become less useful over time. It is now better for Spegel to immediately return content if it exists locally. In addition, the logic for determining the client IP was unreliable and often failed depending on the CNI in use. Overall, the feature added little value and mostly caused confusion.
