---
title: Multi-tenant Credentials
weight: 3
---

Kubernetes will by default skip reaching out to the upstream registry if the image required is present on the node. This speeds pod startup time as it avoids an extra HTTP request. For public images and single tenant clusters, this is not a problem and the preferred behavior. However it can become problematic in multi-tenant clusters using images requiring credentials to pull. In these scenarios one tenant could in theory use another tenants image if they are able to schedule a pod on a node that has that image cached.

Spegel exacerbates this problem as it exposes every nodes image cache to the entire cluster, removing the need to schedule on the correct node. All that is needed is knowledge of the image name and tag. The suggested solution by Kubernetes is to enable the [AlwaysPullImages](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#alwayspullimages) admission controller. This ensures that every scheduled pod will need to prove that its credentials are valid for the given image. The downside is that it puts the registry in the critical path of and container creation event. If the registry is down the container will not be created, even if the image is cached on the node.

This approach does not work with Spegel, or any other registry mirror for that matter. HTTP requests to mirror registries do not include any credentials, so even if the mirror could verify the credentials it can't. Meaning that any request to a mirror would always succeed as long as the mirror has that image cached. Circumventing the solution that always pulling was supposed to solve.

## Admission Enforcement

In many multi-tenant clusters the source of images per tenant is well-known. Either all images are pulled from a unique registry or repository within the same registry per tenant. This means that we don't need to verify that credentials are valid at every pull. Instead this can be enforced before a pod is created, enforcing use of image names per namespaces. For example the namespace `tenant-a` will only be able to create pods using images from `example.com/tenant-a` while namespace `tenant-b` will only be able to pull from `example.com/tenant-b`. Solutions like [Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/), [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper), and [Kyverno](https://github.com/kyverno/kyverno/) can enforce policies for image sources with very little overhead.

## Kvick Operator

<a href="https://kvick.dev" target="_blank" style="color: inherit; text-decoration: none;">
{{< callout type="info" >}}
  <b>Enterprise Feature</b>
{{< /callout >}}
</a>

Ensuring that only tenants with the correct credentials can use a cached image gets significantly more complicated if the source is dynamic for each tenant. The registry may not be known ahead of time making whitelisting complicated or impossible. In these scenarios the Kvick Operator can be used to enforce image usage by verifying credentials with a fallback ensuring that registry downtime will not affect the ability to create new containers.

By enabling image credentials checks, there has to be proof of access to a valid credential that can access the image. The example below is the simplest which enforces this check on every image and denies pulling of new images if the enforcement with the upstream registry fails. The selector is used to group namespaces where these policies apply.

```yaml
kvick:
  imageCredentials:
    enabled: true
    enforcement: Every
    failover: Deny
    selector:
      - kubernetes.io/metadata.name
```

Enforcement can be changed to be more lenient but efficient if wanted. This is useful to avoid verifying credentials for the same image with different tags.

| Enforcement | Description |
| --- | --- |
| Every | Every new image pull will check with upstream. |
| NewRegistry | New registries used in the namespace group require authentication. |
| NewRepository| New repositories in the namespace group require authentication. |
| NewImage | New images in the namespace group require authentication. |

Along with the enforcement the failover policy can also be tuned when the upstream registry is down.

| Failover | Description |
| --- | --- |
| SameRegistry | Allows new images if the same registry has been used in the namespace group. |
| SameRepository | Allows new images if the same repository has been used in the namespace group. |
| SameImage| Allows new images if the same image has been used in the namespace group. |
| Allow | Allows all images. |
| Deny | Denies all images. |
