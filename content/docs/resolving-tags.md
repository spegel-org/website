---
title: Resolving Tags
weight: 2
---

{{< callout type="important" >}}
  Disabling tag resolution or using k8s-digester does not protect against registry outages. If the upstream registry goes down new pods will not be able to resolve the tag.
{{< /callout >}}

Tags are used by images to create named human readable references to manifest digests. Commonly tags are computed from git tags or commit sha sums. This results in a predicable tag that also changes as new versions of an image is pushed to a registry. While tags makes image references simple it has a big downside. We put a lot of trust in the registry that the digest the tag is resolved to is the truth. In a perfect world we would always use the image digest to reference the top level image manifest, but that is rarlely reality.

Things become increasingly complicated when tags are reused for different versions of the same image. This is especially common with the pracitce of reusing the `latest` tag to indicate the latest version of an image.Out of the box Spegel will attempt to resolve tags to digests. Once an image has been pulled by a reusable tag reference, that tag will resolve to the first digest for as long as the image is present in the cluster.Hindering any updates of the image. There are a couple of different methods for how this problem can be solved.

## Disable Tag Resolution

You can disable Spegel from resolving tags completely. This will force the client to resolve the tag from the upstream registry. Once the tag is resolved the rest of the layers will be pulled from Spegel.

```yaml
spegel:
  resolveTags: false
```

## Disable Specific Tags

If only specific tags are being reused it is possible to only disable resolving those. In the example we disable resolving all `latest` tags.

```yaml
spegel:
  registryFilters:
    - :latest$
```

## K8s Digester

The [k8s-digester](https://github.com/google/k8s-digester) project can be used to mutate pods and enforce all images are referenced by digest. Each time a pod is created k8s-digester will resolve the new digest for a tag if a new version is pushed to the registry. One caveat when deploying k8s-digester is that it will by default modify both pods but also any other parent resource that creates pods. This in turn has the side effect of only setting the digest once when the parent resource is created, and never again. For that reason it is a good idea to modify the mutating webhook to only include pods, that way the digest will be updated every time a new pod is created.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: digester-mutating-webhook-configuration
  labels:
    control-plane: controller-manager
    digester/operation: webhook
    digester/system: "yes"
webhooks:
- name: digester-webhook-service.digester-system.svc
  admissionReviewVersions:
  - v1
  - v1beta1
  clientConfig:
    service:
      name: digester-webhook-service
      namespace: digester-system
      path: /v1/mutate
    caBundle: Cg==
  failurePolicy: Ignore
  namespaceSelector:
    matchLabels:
      digest-resolution: enabled
  reinvocationPolicy: IfNeeded
  rules:
  - resources:
    - pods
    apiGroups:
    - ''
    apiVersions:
    - v1
    operations:
    - CREATE
    - UPDATE
    scope: Namespaced
  sideEffects: None
  timeoutSeconds: 15
```

## Kvick Operator

<a href="https://kvick.dev" target="_blank" style="color: inherit; text-decoration: none;">
{{< callout type="info" >}}
  <b>Enterprise Feature</b>
{{< /callout >}}
</a>

The Kvick Operator configures a mutating webhook that resolves tags to digests for each Pod that is created. In the case where the upstream registry goes down it will use the last resolved digest until the upstream registry becomes available again. This information is also persisted across restarts.

To enable tag resolution in the operator configure the Helm chart with the following values.

```yaml
kvick:
  resolveTags:
    enabled: true
```
