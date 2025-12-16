---
title: Announcing Spegel v0.6.0
slug: spegel-v0.6.0
date: 2025-12-16
params:
  authors:
    - name: Philip Laine
      link: https://github.com/phillebaba
---

The [v0.6.0](https://github.com/spegel-org/spegel/releases/tag/v0.6.0) release of Spegel has been focused on increasing the test coverage and stability of Spegel first and foremost. Integration tests have been refactored and extended to ensure compatibility with both Containerd and Kubernetes. The work made it painfully aware that there was a lack of documentation about which versions are supported. This has been remedied with the expansion of the [compatibility documentation](https://spegel.dev/docs/releases/#dependency-compatibility) alongside matrix integration tests. With the current release the Containerd versions that have been tested are `1.7.x`, `2.0.x`, `2.1.x`, and `2.2.x`. The Kubernetes versions tested are `1.32.x`, `1.33.x`, and `1.34.x`. Worth noting that this is the last release of Spegel which will officially support Containerd `1.7.x` and Kubernetes `1.32.x`. Spegel may still work with older versions in the future but that is coincidental and now verification will be done in the future.

I ([phillebaba](https://github.com/phillebaba)) will be taking some time off from working on Spegel, to prioritize my personal life and focus on work that will allow me to pay the bills. From now until the beginning of February you should not expect any responses in either Slack of GitHub issues. I have also cancelled the community calls until February. My hope is that this release will fix some of the stability issues that have been reported over the last few months. If you are in need of support during this time you can reach out to me at [contact@kvick.dev](contact@kvick.dev) to inquire about support contracts.

I wish you all a happy holidays free from outages! 🎅

## Containerd and Kubernetes Integration Tests

Testing is a critical component of delivering stable software. The challenge Spegel has is that it integrates with other tools that have many actively used versions. Both Containerd and Kubernetes offer different feature sets and behave differently depending on the version. Previously Spegel has depended on the e2e tests with Kind to verify integrations with both. This partially worked but had the downside that we could not easily control the Kubernetes or Containerd version used. Over the last month the tests have been refactored and split into separate Containerd and Kubernetes integration tests. This enables the Kubernetes tests to be simplified which enables it to run faster, while moving some of the complex test cases to the Containerd integration tests.

Doing this also enabled matrix testing. The same tests can now be run against all the officially supported Containerd and Kubernetes versions. The tests take some time to run, so PRs will only run the tests against the latest version. To get full coverage [nightly tests](https://github.com/spegel-org/spegel/blob/main/.github/workflows/nightly.yaml) have been added which will also be run before any release is cut.

The investment in updating the testing has already paid off as it has surfaced two different bugs in the Containerd integration. The hope is that it will ensure more stable releases in the future.

## Sweeping Provider

A critical component of Spegel is the DHT which enables discovery of content on other peers. Each peer advertises the content present locally with a set TTL. This means that the content needs to be re-advertised before the TTL expires. Up until now this process would run on a fixed timer that would advertise all the content in one go. Which would create a spike in CPU and memory usage. Thanks to the great work done by the maintainers of [go-libp2p-kad-dht](https://github.com/libp2p/go-libp2p-kad-dht) there is a new solution called sweeping provider that reduces the resource spikes by batching content advertisement in an intelligent manner. The old solution has been completely replaced by the new provider so expect to see some reduction in top level resource consumption.

## Dual Stack Support

Alongside the new provider implementation some of the address advertisement logic was refactored. A lot of this code has not been touched since the initial implementation. When initially implemented only single stack Kubernetes clusters were considered. That being clusters only using IPv4 or IPv6. While not as common there are people running dual stack clusters, which would at random times not work. To resolve this new address selection logic has been added which is aware of the stack which is locally supported and returns addresses with a priority for IPv6.
