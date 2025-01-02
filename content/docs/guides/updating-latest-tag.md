---
title: Updating latest tag
description: How to update images reusing tags.
---

Reusing the same tag multiple times for different versions of an image is generally a bad idea. The most common scenario is the use of the `latest` tag. This makes it difficult to determine which version of the image is being used. On top of that, the image will not be updated if it is already cached on the node. Some people have chosen to power forward with reusing tags and chosen to instead set the image pull policy to `AlwaysPull`, forcing the image to be updated every time a pod is started. This will however not work with Spegel as the tag could be resolved by another node in the cluster resulting in the same "old" image being pulled. There are two solutions to work around this problem, allowing users to continue with their way of working before using Spegel.

The best solution to this problem is to deploy [k8s-digester](https://github.com/google/k8s-digester) alongside Spegel. It will allow you to enjoy all the benefits of Spegel will continuously updating image tag versions. The way it works is that k8s-digester will, for each pod created, resolve tags to image digests and add them to the image reference. All pods that originally reference images by tag will instead do so with digest. Each time k8s-digester will resolve the new digest for a tag if a new version is pushed to the registry. Spegel will be able to continue distributing images if the external registry became unavailable. The reason this works is that the mutating webhook is configured to ignore errors, and instead, Spegel will be used to resolve the tag to a digest.

One caveat when deploying k8s-digester is that it will by default modify both pods but also any other parent resource that creates pods. This in turn has the side effect of only setting the digest once when the parent resource is created, and never again. For that reason it is a good idea to modify the mutating webhook to only include pods, that way the digest will be updated every time a new pod is created.

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
  failurePolicy: Ignore # kpt-set: ${failure-policy}
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

The second option, which should be used only if using k8s-digester is not possible, is to disable tag resolving altogether in Spegel. There are two options when doing this. It can either be disabled only for `latest` tags or for all tags. This can be done by changing the Helm charts values from their defaults.

```yaml
spegel:
  resolveTags: false
  resolveLatestTag: false
```

Please note that this does however remove Spegel's ability to protect against registry outages for any images referenced by tags.
