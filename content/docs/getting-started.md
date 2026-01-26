---
title: Getting Started
weight: 1
---

Before deploying Spegel read the compatibility section to make sure that the Kubernetes flavor of your choosing is supported or requires specific configuration to work.

## Deploying

Use the Helm chart to deploy Spegel into your Kubernetes cluster. Refer to the [Helm Chart](https://github.com/spegel-org/spegel/tree/main/charts/spegel) for detailed configuration documentation.

### CLI

To deploy Spegel with the Helm CLI run the command.

```shell
helm upgrade --create-namespace --namespace spegel --install spegel oci://ghcr.io/spegel-org/helm-charts/spegel
```

### Flux

To deploy Spegel with Flux commit the configuration.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: spegel
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: spegel
  namespace: spegel
spec:
  type: "oci"
  interval: 5m0s
  url: oci://ghcr.io/spegel-org/helm-charts
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: spegel
  namespace: spegel
spec:
  interval: 1m
  chart:
    spec:
      chart: spegel
      interval: 5m
      sourceRef:
        kind: HelmRepository
        name: spegel
```

## Compatibility

Currently, Spegel only works with Containerd, in the future other container runtime interfaces may be supported. Spegel relies on [Containerd registry mirroring](https://github.com/containerd/containerd/blob/main/docs/hosts.md#cri) to route requests to the correct destination.
This requires Containerd to be properly configured, if it is not Spegel will exit. First of all the registry config path needs to be set, this is not done by default in Containerd. Second of all discarding unpacked layers cannot be enabled.
Some Kubernetes flavors come with this setting out of the box, while others do not. Spegel is not able to write this configuration for you as it requires a restart of Containerd to take effect.

```toml {filename="/etc/containerd/config.toml"}
version = 2

[plugins."io.containerd.grpc.v1.cri".registry]
   config_path = "/etc/containerd/certs.d"
[plugins."io.containerd.grpc.v1.cri".containerd]
   discard_unpacked_layers = false
```

Spegel has been tested on the following Kubernetes distributions for compatibility. Green status means Spegel will work out of the box, yellow will require additional configuration, and red means that Spegel will not work.

| Distribution | Status |
| :--- | :---: |
| [AKS](https://azure.microsoft.com/en-us/products/kubernetes-service)   | :green_circle:  |
| [Minikube](https://minikube.sigs.k8s.io/docs/)                         | :green_circle:  |
| [Kapsule](https://www.scaleway.com/en/kubernetes-kapsule/)             | :green_circle:  |
| [NKP](https://www.nutanix.com/products/kubernetes-management-platform) | :green_circle:  |
| [EKS](https://aws.amazon.com/eks/)                                     | :yellow_circle: |
| [K0S](https://k0sproject.io/)                                          | :yellow_circle: |
| [K3S](https://k3s.io/) and [RKE2](https://docs.rke2.io/)               | :yellow_circle: |
| [Kind](https://kind.sigs.k8s.io/)                                      | :yellow_circle: |
| [Talos](https://www.talos.dev/)                                        | :yellow_circle: |
| [VKE](https://www.volcengine.com/product/vke)                          | :yellow_circle: |
| [GKE](https://cloud.google.com/kubernetes-engine)                      | :red_circle:    |
| [DigitalOcean](https://www.digitalocean.com/products/kubernetes)       | :red_circle:    |

### EKS

Discard unpacked layers is enabled by default, meaning that layers that are not required for the container runtime will be removed after consumed. This needs to be disabled as otherwise all of the required layers of an image would not be present on the node. AL2023 based EKS AMIs allows customization of Containerd using [nodeadm configuration](https://awslabs.github.io/amazon-eks-ami/nodeadm/).

```yaml
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="BOUNDARY"

--BOUNDARY
Content-Type: application/node.eks.aws

---
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  containerd:
    config: |
      [plugins."io.containerd.cri.v1.images".registry]
        config_path = "/etc/containerd/certs.d"
      [plugins.'io.containerd.cri.v1.images']
        discard_unpacked_layers = false

--BOUNDARY--
```

### K0S

As K0S packages its own Containerd some paths will be different than the standard defaults. First of all the Containerd configuration needs to be appended to. The following configuration must be written to `/etc/k0s/containerd.d/spegel.toml` before K0S starts for the configuration to be picked up.

```toml {filename="/etc/k0s/containerd.d/spegel.toml"}
[plugins."io.containerd.grpc.v1.cri".registry]
   config_path = "/etc/containerd/certs.d"
[plugins."io.containerd.grpc.v1.cri".containerd]
   discard_unpacked_layers = false
```

After K0S has started Spegel can be installed with the slight modified values as the Containerd socket and content path will be different.

```yaml
spegel:
  containerdSock: "/run/k0s/containerd.sock"
  containerdContentPath: "/var/lib/k0s/containerd/io.containerd.content.v1.content"
```

### K3S and RKE2

K3S and RKE2 embeds Spegel, refer to their [documentation](https://docs.k3s.io/installation/registry-mirror?_highlight=spegel) for deployment information.

### Kind

Spegel uses Kind for its end-to-end tests.

Discard unpacked layers is enabled by default, meaning that layers that are not required for the container runtime will be removed after consumed.
This needs to be disabled as otherwise all of the required layers of an image would not be present on the node.

In order to be more like a "real" Kubernetes cluster, you may want to set [Containerd's metadata sharing policy](https://github.com/containerd/containerd/blob/main/docs/ops.md#bolt-metadata-plugin) to `isolated`.

```yaml
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry]
    config_path = "/etc/containerd/certs.d"
  [plugins."io.containerd.grpc.v1.cri".containerd]
    discard_unpacked_layers = false
  [plugins."io.containerd.metadata.v1.bolt"]
    content_sharing_policy = "isolated"
```

### Talos

Talos will by default discard unpacked layers, which has to be disabled with a machine configuration.

```yaml
machine:
  files:
    - path: /etc/cri/conf.d/20-customization.part
      op: create
      content: |
        [plugins."io.containerd.cri.v1.images"]
          discard_unpacked_layers = false
```

Talos also uses a different path as its Containerd registry config path.

```yaml
spegel:
  containerdRegistryConfigPath: /etc/cri/conf.d/hosts
```

Talos comes with Pod Security Admission [pre-configured](https://www.talos.dev/latest/kubernetes-guides/configuration/pod-security/). The default Talos security profile is too restrictive to allow Spegel to operate, therefore it's necessary to adjust the security profile of the Spegel namespace.

```shell
kubectl label namespace spegel pod-security.kubernetes.io/enforce=privileged
```

### VKE

VKE CNI Cello [don't support](https://github.com/volcengine/cello/issues/15) setting hostPorts at the same range of `net.ipv4.ip_local_port_range`.
You can view the exact range by running `sysctl net.ipv4.ip_local_port_range` on your node.

To workaround this issue, you need to set a custom hostPort when deploying spegel.
This can easily be done by setting hostPort in the helm chart.

```yaml
service:
  registry:
    hostPort: 3020
```

### GKE

GKE does not set the registry config path in its Containerd configuration. On top of that it uses the old mirror configuration for the internal mirroring service.

### DigitalOcean

DigitalOcean does not set the registry config path in its Containerd configuration.
