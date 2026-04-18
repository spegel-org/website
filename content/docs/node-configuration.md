---
title: Node Configuration
weight: 2
---

Spegel depends on certain mirror configuration on each node to function. This configuration makes sure that requests go to the local Spegel instance on the node instead of the upstream registry. Out of the box, Spegel writes this configuration using an init container. Ensuring the configuration is there before the registry starts. While the solution is simple and frictionless it has some downsides. One of them being that the bootstrap process relies on the Spegel image to write the configuration in the first place. Creating a chicken and egg situation.

On top of that when a new node is created, a race occurs for all the pending containers to start running. Pods created by daemonsets will be competing with other pending pods waiting for a node with available resources. While we want pods to start up as quickly as possible there are certain pods that should be ready before any other pod is scheduled on the node. If Spegel is not running and ready on the node it is not able to configure the mirror configuration and serve successive images from within the cluster. This will result in images used by pods scheduled before being pulled from the upstream registry which is both wasteful and slow. While priority classes helps with scheduling priority it does not ensure that Spegel is ready before allowing other pods to schedule. There are two alternative options to fix these downsides.

## User Data

Self hosted users and certain cloud providers allow for custom configuration of the VM that is applied during the boot process and before the VM is ready to run containers. This is a perfect opportunity to configure the mirror configuration. Removing the need to have Spegel do it in an init container. The mirror host has to be the node ip and not `localhost` or `127.0.0.1`.

```toml {filename="/etc/containerd/certs.d/_default/hosts.toml"}
[host.'http://${NODE_IP}:30020']
capabilities = ['pull', 'resolve']
dial_timeout = '200ms'
```

The init container writing the Containerd mirror configuration is no longer needed and can be disabled in the Helm chart.

```yaml
spegel:
  containerdMirrorAdd: false
```

## Kvick Operator

<a href="https://kvick.dev" target="_blank" style="color: inherit; text-decoration: none;">
{{< callout type="info" >}}
  <b>Enterprise Feature</b>
{{< /callout >}}
</a>

The Kvick operator ensures that mirror configuration is written and Spegel is ready before any other pods can be scheduled on the node. It does this by tainting new nodes that have not been configured, and using existing base images on the node to write the configuration we can remove any need for external registries to bootstrap the configuration. Once that is done the Spegel pod is started and once it is ready the taint is removed from node allowing pending pods to be scheduled. This solution removes any external image pulls during node provisioning while also ensuring stability by functioning even in airgapped environments.

Enable node readiness taints in the operator to ensure Spegel is running first.

```yaml
kvick:
  nodeReadiness:
    enabled: true
```

The init container writing the Containerd mirror configuration is no longer needed and can be disabled in the Helm chart.

```yaml
spegel:
  containerdMirrorAdd: false
```
