---
title: Host Installation
weight: 3
---

Spegel can run directly on a Linux host alongside containerd, without
Kubernetes, using the HTTP bootstrap to discover peers. This page describes
the end-to-end install on two or more hosts.

## Prerequisites

- Linux with `systemd` as PID 1.
- containerd 2.x with the CRI plugin enabled. Older 1.7.x releases work
  provided `config_path` is set under the CRI registry config; the examples
  below assume containerd 2.x layout.
- Each peer must be able to reach the other peers on the ports listed in
  the [Network ports](#network-ports) table.

## Install the binary

Download the latest release from the
[releases page](https://github.com/spegel-org/spegel/releases) and place
the binary at `/usr/local/bin/spegel`.

```shell
install -m 0755 spegel /usr/local/bin/spegel
```

## Systemd unit

Drop the following unit at `/etc/systemd/system/spegel.service`.

```ini
[Unit]
Description=Spegel peer-to-peer image cache
Documentation=https://spegel.dev
After=containerd.service network-online.target
Requires=containerd.service
Wants=network-online.target

[Service]
Type=simple
EnvironmentFile=/etc/spegel/peer.env
ExecStart=/usr/local/bin/spegel registry \
  --bootstrap-kind=http \
  --http-bootstrap-addr=:5002 \
  --http-bootstrap-peer=${SPEGEL_HTTP_BOOTSTRAP_PEER} \
  --mirrored-registries https://ghcr.io https://registry.k8s.io https://docker.io
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Adjust the `--mirrored-registries` list to the set of upstream registries
that should be peer-cached on this host.

## Bootstrap peers

The unit reads `SPEGEL_HTTP_BOOTSTRAP_PEER` from
`/etc/spegel/peer.env`. On each host, point the variable at one of the
other peers' HTTP bootstrap endpoint. For a two-host cluster
(`<NODE_A_IP>` and `<NODE_B_IP>`):

On `<NODE_A_IP>`:

```shell {filename="/etc/spegel/peer.env"}
SPEGEL_HTTP_BOOTSTRAP_PEER=http://<NODE_B_IP>:5002
```

On `<NODE_B_IP>`:

```shell {filename="/etc/spegel/peer.env"}
SPEGEL_HTTP_BOOTSTRAP_PEER=http://<NODE_A_IP>:5002
```

Any reachable peer works; the bootstrap endpoint is symmetric. With three
or more peers, pointing each at the next host in a ring is sufficient.

## Configure containerd

Generate the containerd registry configuration on every host so containerd
routes mirrored pulls through the local Spegel instance.

```shell
/usr/local/bin/spegel configuration \
  --mirror-targets http://127.0.0.1:5000 \
  --mirrored-registries https://ghcr.io https://registry.k8s.io https://docker.io
```

This writes one `hosts.toml` per registry under
`/etc/containerd/certs.d/<registry>/`. Make sure containerd's CRI registry
config points at that directory:

```toml {filename="/etc/containerd/config.toml"}
[plugins.'io.containerd.cri.v1.images'.registry]
  config_path = '/etc/containerd/certs.d'
```

Reload containerd so the new mirror configuration takes effect, then start
Spegel.

```shell
systemctl reload-or-restart containerd
systemctl enable --now spegel.service
```

## Network ports

| Port | Protocol | Purpose |
| ---- | -------- | ------- |
| 5000 | TCP | OCI registry endpoint that the local containerd dials as a mirror. |
| 5001 | TCP + UDP | libp2p router used for peer discovery and content lookups. |
| 5002 | TCP | HTTP bootstrap endpoint reachable by every other peer. |
| 9090 | TCP | Prometheus metrics endpoint. |

5000 and 9090 only need to be reachable from `127.0.0.1`. 5001 and 5002
must be reachable from every other peer.

## Operational notes

- `Restart=always` keeps Spegel alive across crashes; after a host reboot
  systemd starts containerd first (`Requires=containerd.service`) and then
  Spegel, which means a freshly booted host rejoins the peer set
  automatically.
- If a configured bootstrap peer is unreachable when Spegel starts, the
  log contains repeated `failed to run bootstrap` entries until either
  the peer becomes reachable or the bootstrap timeout elapses. Spegel
  then exits and is restarted by systemd; meanwhile pulls fall back to
  the upstream registry, so the only visible effect is reduced cache
  hits.
- Point Spegel at a non-default containerd content path with
  `--containerd-content-path`. The flag must match containerd's `root`
  setting; otherwise Spegel cannot enumerate locally stored blobs and
  will only serve content it has fetched itself.

## Verify

Confirm the peer-discovery handshake completed on each host:

```shell
journalctl -u spegel.service --no-pager | grep "bootstrap completed connectivity is reached"
```

Trigger a pull on one host, then confirm the mirror counter incremented:

```shell
crictl pull ghcr.io/spegel-org/benchmark:v2-10MB-4
curl -sf http://127.0.0.1:9090/metrics | grep spegel_mirror_requests_total
```

A non-zero `spegel_mirror_requests_total{cache="hit"}` confirms the pull
was served from a peer rather than from the upstream registry.
