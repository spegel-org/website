---
title: Releases
type: docs
weight: 7
---

Spegel is distributed as both precompiled standalone binaries and Docker images. Releases are provided for the amd64, arm64, and armv7 architectures.

The Docker images are published to the GitHub Container Registry within the spegel-org GitHub organization and can be accessed at the [Spegel package](https://github.com/spegel-org/spegel/pkgs/container/spegel).

Release binaries are available on the [Spegel releases page](https://github.com/spegel-org/spegel/releases), where you can download the version suited to your platform.

## Versioning 

Spegel follows the [Semantic Versioning (SemVer)](https://semver.org/) convention, using the pattern `vX.Y.Z` where:

* `X` represents the major version.
* `Y` represents the minor version.
* `Z` represents the patch version.

This versioning scheme clearly communicates change impact. Increments to the major version indicate backward-incompatible changes, minor version increments indicate backward-compatible feature additions, and patch increments indicate backward-compatible bug fixes. Note that versions before `v1.0.0` may include breaking changes as the public API is considered unstable.

## Dependency Compatibility

Spegel integrates with two primary dependencies: Kubernetes and Containerd. These projects maintain independent release cadences, and Spegel strives to support a broad range of their versions. At each release, Spegel is tested against available versions in accordance with its defined support policy.

Spegel follows the widely adopted Kubernetes (N-2) [version compatibility policy](https://kubernetes.io/releases/version-skew-policy/). This means Spegel supports the latest stable Kubernetes minor version (N) as well as the two immediately preceding minor versions (N-1 and N-2).

For Containerd, Spegel supports versions from 1.7 onward that are under [active maintenance](https://containerd.io/releases/#current-state-of-containerd-releases). Compatibility with actively supported Containerd versions is maintained to the best extent possible. Once a Containerd version reaches end-of-life, backwards compatibility may be removed.

Commercial support options are available for users requiring assistance with older Kubernetes or Containerd versions. Please refer to the [Support](/support) page for more information.
