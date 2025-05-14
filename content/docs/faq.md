---
title: FAQ
type: docs
weight: 6
---

## Can I use Spegel in production?

Spegel is being used by multiple users in production for over a year without any major issues. The great thing is that pulling images would not stop working if you for some reason would find an issue with Spegel.
A fallback to the original registry will always occur if Spegel can't be reached or serve the requested image.

## What performance increase can I expect with Spegel?

Read the [benchmark documentation](/docs/benchmark/) for information of expected gains.

## How do I know that Spegel is working? 

Spegel is meant to be a painless experience to install, meaning that it may be difficult initially to know if things are working or not. Simply put a good indicator that things are working is if all Spegel pods have started and are in a ready state.
Spegel does a couple of checks on startup to verify that any required configuration is correct, if it is not it will exit with an error. While it runs it will log all received requests, both those it mirrors and it serves.

An incoming request to Spegel that is mirrored will receive the following log.

```json
{"level":"info","ts":1692304805.9038486,"caller":"gin@v0.0.9/logger.go:53","msg":"","path":"/v2/library/nginx/blobs/sha256:1cb127bd932119089b5ffb612ffa84537ddd1318e6784f2fce80916bbb8bd166","status":200,"method":"GET","latency":0.005075836,"ip":"172.18.0.5","handler":"mirror"}
```

While the Spegel instance on the other end will log.

```json
{"level":"info","ts":1692304805.9035861,"caller":"gin@v0.0.9/logger.go:53","msg":"","path":"/v2/library/nginx/blobs/sha256:1cb127bd932119089b5ffb612ffa84537ddd1318e6784f2fce80916bbb8bd166","status":200,"method":"GET","latency":0.003644997,"ip":"172.18.0.5","handler":"blob"}
```

## Will image pulls break or be delayed if a spegel instance fails or is removed?

Spegel acts as a best-effort cache and the worst-case scenario is always that images are pulled from the upstream registry (e.g. Docker Hub).

However, should a spegel instance fail (perhaps because the node died), there will be a time interval when its images remain advertised. Currently, spegel advertises images with a TTL of 10 minutes. Other spegel peers may try to forward requests to the failed instance, delaying the response to the pulling client. In benign scenarios, this delay is the length of an intra-cluster round trip (the HTTP request and an ICMP unreachable response), likely <1ms. Of course, there are less benign scenarios (e.g. inter-node packet loss) where no replies will come back and spegel's forwarder will eventually time out before moving on to the next available peer. Spegel uses the standard library's httputil.ReverseProxy to forward requests, which in turn depends on DefaultTransport to decide how long to wait before giving up.

Please note that a client is likely to request several layers in parallel and in many cases the advertising instances will have a similar routing distance, so spegel will spread its forwards across those instances. Thus, the benign scenario is unlikely to impact pod startup time. Only when the routing distance is different (e.g. edge locations) or when an image dominated by one large layer is affected is pod startup time materially increased.

## Why am I not able to pull the new version of my tagged image?

Read the [updating latest tag](/docs/guides/updating-latest-tag/) guide.

## Why am I able to pull private images without image pull secrets?

An image pulled by a Kubernetes node is cached locally on disk. Meaning that other pods running on the same node that require the same image do not have to pull the same image again. Spegel relies on this mechanism to be able to distribute images.
This may however not be a desirable feature when running a multi-tenant cluster where private images are pulled using credentials. In this scenario, only those pods with the correct credentials would be able to use the image.
Ownership of private images has been an issue for a long time in Kubernetes as indicated by the unresolved issue https://github.com/kubernetes/kubernetes/issues/18787 created back in 2015. The short answer is that a good solution does not exist, with or without Spegel.
The current [suggested solution](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#alwayspullimages) is to enforce an `AlwaysPull` image policy for private images that require authentication. Doing so will force a request to the registry to
validate the digest or resolve the tag. This request will only succeed with the proper authentication. This is a mediocre solution at best as it creates a hard dependency on the external registry, meaning the pod will not be able to start even if the image is cached on the node.

This solution does however not work when using Spegel, instead, Spegel may make the problem worse. Without Spegel an image that would want to use a private image, it does not have access to would have to be scheduled on a node that has already pulled the image.
With Spegel that image will be available to all nodes in the cluster. Currently, a good solution for Spegel does not exist. There are two reasons for this. The first is that credentials are not included when pulling an image from a registry mirror, a good choice as doing so would mean sharing credentials with third parties.
Additionally, Spegel would have no method of validating the credentials even if they were included in the requests. So for the time being if you have these types of requirements Spegel may not be the choice for you.

## How do I use Spegel in conjunction with another registry cache?

Spegel can be used with other registry caches in cases where the best effort caching offered by Spegel is not enough. In these situations, if the image is not cached within the cluster the image should be pulled from the secondary cache.
This is configured by adding the domain of the registry to the `additionalMirrorRegistries` list in the Helm values. Registries added to this list will be included in the mirror configuration created by Spegel.

```yaml
spegel:
  additionalMirrorRegistries:
    - https://zot.example.com
```

## Why is my node running out of disk space?

By default the kubelet on every node is configured to [garbage collect](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#containers-images) unused images when the disk space starts to run out. Some Kubernetes clusters come with image garbage collection disabled by default. This can cause a nodes disk to fill up quickly, especially on nodes with small disks to begin with. Spegel does not have a built in garbage collection instead it depends completely on the kubelt garbage collection being properly configured.

## What should I do if other pods are scheduled on new nodes before Spegel?

The Kubernetes scheduler will assign pods to nodes as soon as the node reports as ready. This causes a race to schedule and start Spegel on any new node, before other pods are scheduled. If this does not happen the mirror configuration will not be written to the node before the node starts pulling images for other new pods scheduled on it. Defeating the purpose of Spegel in this scenario. This problem is not unique to Spegel, but is a wider problem in Kubernetes for critical daemonsets. There are two closed KEPs [#1003](https://github.com/kubernetes/enhancements/pull/1003) and [#75890](https://github.com/kubernetes/kubernetes/issues/75890) which attempted to solve this without being accepted.

The best solution to address this problem currently is to use [nidhogg](https://github.com/pelotech/nidhogg) to taint nodes which are not running pods from specific daemonsets. It implements for the most part the features suggested in both KEPs. Ensuring that all image pulls go through Spegel, even on new nodes.

## Can I deploy multiple Spegel clusters?

In certain situations deploying multiple separate Spegel clusters is beneficial. For example when a Kubernetes cluster spans multiple regions, it may be beneficial to limit nodes too only pull images from within the same region. Spegel can be deployed multiple times by simply using different names for each Helm deployment and setting a unique node selector. It is important to note that the node port service needs a unique port per deployment. Spegel will then only deploy on nodes with the matching labels and elect a unique leader for each deployment of Spegel. As the two Spegel clusters will never communicate they will not be able to discover layers outside of their own region, limiting requests to their specific region.

```bash
helm upgrade --create-namespace --namespace spegel --install spegel-one oci://ghcr.io/spegel-org/helm-charts/spegel --set "nodeSelector.group=one" --set "service.registry.nodePort=30021"
helm upgrade --create-namespace --namespace spegel --install spegel-two oci://ghcr.io/spegel-org/helm-charts/spegel --set "nodeSelector.group=two" --set "service.registry.nodePort=30022"
```

## Why are default memory requests and limits set in the Helm chart?

Linux file reading relies on the VFS cache, which keeps file chunks in memory for efficiency. In a containerized environment, VFS cache usage contributes to the container's memory consumption. Since VFS cache is only cleared under memory pressure, Spegel may appear to consume increasing amounts of memory until it reaches the available memory on the node. To prevent excessive memory usage, the Helm chart sets a default memory limit of 128Mi, ensuring that the cache is cleared when necessary.
