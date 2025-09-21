---
title: Announcing Spegel v0.4.0
slug: spegel-v0.4.0
date: 2025-09-21
params:
  authors:
    - name: Philip Laine
      link: https://github.com/phillebaba
---

Summer has come and gone, so it has been a while since the last release of Spegel. This has given a lot of time for refactoring and general cleanup of tests and other functionality. A lot of this work is a requirement to implement more complicated features in the future. The [v0.4.0](https://github.com/spegel-org/spegel/releases/tag/v0.4.0) release still contains some changes that are worth highlighting, especially as it contains a deprecation which will be removed in a future release.

## OCI Volumes

While not a new feature per se, checks have been added to the e2e tests to verify that OCI volumes are compatible with Spegel. The feature is implemented using the existing CRI API, which means that no additional changes were required in Spegel to support the feature. Officially supporting OCI volumes opens up a lot of doors, especially for AI use cases, as the artifacts tend to be large and pulling within the cluster will be faster than egressing to an external registry. The OCI volume feature is still in beta hence disabled by default. Support for OCI volumes will become more relevant as the feature graduates allowing other projects like [kserve](https://github.com/kserve/kserve) start supporting the volume type, along with greater community adoption. 

## Resolve Latest Tag Deprecation

I challenge some users face when reusing image tags together with Spegel, is how to update the digest which the tag resolves to. While it can be debated whether or not reusing tags is a good pracitce, it is something that parts of the community does as part of their release strategy. While the suggested solution in the [Updating Latest Tag](https://spegel.dev/docs/guides/updating-latest-tag/) guide works for most, there are those that are not able to use mutating webhooks. Early on the `resolveLatestTag` flag was added to stop Spegel from resolving any image using the `latest` tag while resolving all other tags. This works well as long as the tag that is being reused is `latest`, but does not work with any other tag name.

There was a need to support more generic use cases which allows ignoring other tag names or even pattern matching. A new configuration parameter called `registryFilters` was added by [@pepordev](https://github.com/pepordev) in [#1003](https://github.com/spegel-org/spegel/pull/1003/files) that allows regex pattern filtering of registry requests. This allows a lot of customizability in the types of images that need to be ignored by Spegel, not just only specific tag names. This would allow disabling tag resolution for the `dev` tag for images from `docker.io`.

```yaml
registryFilters:
  - ^docker\.io/.+:dev$
```

This new feature means that `resolveLatestTag` is no longer needed as the same functionality can be expressed using the new feature. The old parameter is now deprecated and will be removed in the next release. If you are using it please update your configuration. The following registry filter will achieve the same behavior as the flag did.

```yaml
registryFilters:
  - :latest$
```

## Verifying Spegel

It has been a long time coming, but we finally have a reasonable method of verifying Spegel's core functionality after an installation. Previous documentation had suggested looking at request logs to determine if Spegel is properly mirroring requests or not. Which not so surprisingly was a horrible UX, especially for new users. The distributed nature of Spegel along with the requirement to keep Spegel stateless has made it challenging to implement a good solution. The changes done in [#986](https://github.com/spegel-org/spegel/pull/986) changed the default logging verbosity of request logs to be opt in. Removing the ability to verify request logs easily, forcing the implementation of a new solution.

The debug web view is now enabled by default, serving the page on the same port as metrics and tracing. The debug web view has a new metric called last successful mirror which indicates if the Spegel instance has successfully served content. If the value is pending, it means that this has yet to occur.

![Last Successful Mirror](/blog/spegel-v0-4-0/last-successful-mirror.png)

There is now a new [Verifying Spegel Is Working](https://spegel.dev/docs/guides/verifying-spegel-is-working/) guide which walks through how to force an image pull on a specific node and then verify that it was sucsefully mirrored by Spegel.

## OCI refactoring

The proxy component of the registry has undergone some refactoring to be more OCI aware. This was necissary to implement current and future features. This release contains two fixes that were only possible with the refactoring. Previously if a mirror request fails partailly through downloading a layer, the request would be retried with a new mirror. The problem with this implementation is that the request would be retried from the beginning, causing the served content to be incorrect. Being aware of the request type and optional range requests enables retriable requests that starts after the last byte that was written.

Additionally timeouts have been added to all requests other than GET requetst for blobs. Networking is assumed to be reasonably fast within private networking, allowing the proxy to be slightly more aggressive with it's timeout. All forwarded HEAD requests have a timeout of 1 second while GET requests have a timeout out of 2 seconds. The goal is to move on to the next mirror quickly rather than indefinatly waiting for a response from a Spegel instance that is either not running or is already overloaded. The only forwarded request that does not have a timeout set is GET requests for blobs. Figuring out a suitable timeout for the request is a bit more complicated as the content served can be anywhere between 1 Kilobyte to 1 Terabyte. There is ongoing work to implement a timeout strategy for these requests to avoid having requests wait forever.
