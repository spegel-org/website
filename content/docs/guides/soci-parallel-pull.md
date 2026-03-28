---
title: Using Spegel with AWS SOCI Snapshotter
description: How to configure Spegel with the AWS SOCI Snapshotter in parallel pull mode.
---

Spegel can be used alongside the [AWS SOCI Snapshotter](https://github.com/awslabs/soci-snapshotter) when SOCI is configured in **parallel pull mode**. This guide explains the requirements, configuration, and trade-offs.

## Background

Spegel serves OCI image content (manifests, configs, and layer blobs) from the containerd content store to peer nodes in the cluster. For this to work, image layer blobs must be physically present in the content store on disk.

SOCI Snapshotter supports two pull modes:

- **Lazy pull** (default): Layers are fetched on-demand via FUSE as the container accesses files. Layer blobs are never written to the containerd content store. **This mode is not compatible with Spegel.**
- **Parallel pull**: Full layer blobs are downloaded and unpacked upfront. When configured correctly, blobs are written to the containerd content store, making them available for Spegel to distribute to peer nodes.

## Requirements

SOCI parallel pull mode is compatible with Spegel when **all** of the following are true:

1. `pull_modes.parallel_pull_unpack.enable` is set to `true` in the SOCI config.
2. `pull_modes.parallel_pull_unpack.discard_unpacked_layers` is set to `false` in the SOCI config.
3. `content_store.type` is set to `"containerd"` in the SOCI config.

The critical setting is `discard_unpacked_layers`. When set to `true`, SOCI discards compressed layer blobs after unpacking them into the snapshotter directory. This leaves nothing for Spegel to serve. When set to `false`, SOCI pushes the full compressed blob to the containerd content store, making it available at the standard path that Spegel reads from.

{{% details title="Why does lazy pull not work?" closed="true" %}}

In lazy pull mode, SOCI mounts a FUSE filesystem backed by HTTP range requests to the remote registry. Layer blobs are never fully downloaded or stored in the containerd content store. Since Spegel reads and serves content from the content store, there is nothing available for it to distribute.

{{% /details %}}

## SOCI Configuration

A minimal SOCI configuration for use with Spegel:

```toml
[content_store]
type = "containerd"

[pull_modes.parallel_pull_unpack]
enable = true
discard_unpacked_layers = false
max_concurrent_downloads = -1
max_concurrent_downloads_per_image = 32
max_concurrent_unpacks = -1
max_concurrent_unpacks_per_image = 12

[pull_modes.parallel_pull_unpack.decompress_streams."gzip"]
path = "/usr/bin/unpigz"
args = ["-d", "-c"]
```

The concurrency settings (`max_concurrent_downloads_per_image`, `max_concurrent_unpacks_per_image`, etc.) can be tuned for your instance type and storage performance.

## Containerd Configuration

When using containerd v2.x with SOCI as a proxy snapshotter, the containerd configuration should register SOCI and set the registry config path that Spegel will write mirror configuration to:

```toml
[plugins."io.containerd.cri.v1.images"]
snapshotter = "soci"
disable_snapshot_annotations = false

[plugins."io.containerd.cri.v1.images".registry]
config_path = "/etc/containerd/certs.d"

[proxy_plugins.soci]
type = "snapshot"
address = "/run/soci-snapshotter-grpc/soci-snapshotter-grpc.sock"

[proxy_plugins.soci.exports]
root = "/var/lib/soci-snapshotter-grpc"
```

The `proxy_plugins.soci.exports.root` setting is used by Kubernetes for snapshotter disk accounting and garbage-collection related reporting.

{{% details title="Note on containerd's discard_unpacked_layers" closed="true" %}}

Containerd has its own `discard_unpacked_layers` setting, separate from SOCI's. In a SOCI parallel pull setup, SOCI handles the pull and writes blobs to the content store directly via the containerd content API. The containerd-level setting controls containerd's own unpack behavior, which is not used when SOCI manages pulling.

Spegel skips the containerd configuration verification on containerd v2.x, so this setting does not cause a startup failure.

{{% /details %}}

## Spegel Helm Values

Ensure the Spegel Helm values match your containerd layout:

```yaml
spegel:
  containerdSock: "/run/containerd/containerd.sock"
  containerdNamespace: "k8s.io"
  containerdRegistryConfigPath: "/etc/containerd/certs.d"
  containerdContentPath: "/var/lib/containerd/io.containerd.content.v1.content"
```

If your containerd root is symlinked to another volume (e.g., NVMe storage at `/opt/dlami/nvme/containerd/data-root`), the default `containerdContentPath` value will work as long as `/var/lib/containerd` is a symlink to that location. Symlinks are resolved transparently by the filesystem.

## Kubelet Image Service Endpoint

When SOCI is configured as the image service, kubelet is typically configured to use SOCI's gRPC socket for image operations (so that SOCI handles pulls and ECR credential passthrough). This is done via kubelet configuration:

```json
{
    "imageServiceEndpoint": "unix:///run/soci-snapshotter-grpc/soci-snapshotter-grpc.sock"
}
```

This does not conflict with Spegel. Spegel configures containerd's registry mirrors, and SOCI proxies registry requests through containerd, so the mirror configuration takes effect.

## Disk Space Trade-off

With `discard_unpacked_layers = false`, each image layer is stored twice on disk:

1. **Compressed blob** in the containerd content store, used by Spegel for P2P distribution.
2. **Unpacked filesystem** in the snapshotter directory, used by containers at runtime.

This roughly doubles the per-image disk footprint compared to `discard_unpacked_layers = true`. Plan node storage capacity accordingly. Containerd's garbage collector will clean up blobs when images are deleted.

## Compatibility Matrix

| SOCI Pull Mode | `discard_unpacked_layers` | Content Store Type | Spegel Compatible |
|----------------|---------------------------|--------------------|-------------------|
| Lazy pull      | N/A                       | Any                | No                |
| Parallel pull  | `true`                    | Any                | No                |
| Parallel pull  | `false`                   | `soci`             | No                |
| Parallel pull  | `false`                   | `containerd`       | **Yes**           |
