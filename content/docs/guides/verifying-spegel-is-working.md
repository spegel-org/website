---
title: Verifying Spegel Is Working
description: How to verify that Spegel is working.
weight: 1
---

Spegel has a stateless distributed design that can make it hard to understand when it is functioning, especially after the first installation. The fallback mechanism means that if Spegel is not able to serve the image the pull will fallback to the upstream registry. Silencing any potential issues that may exist. Spegel does a couple of checks on startup to verify that any required configuration is correct, if it is not it will exit with an error. There may however exist other issues that Spegel may not catch, so a manual validation of the installation can be at time valuable.

As Spegel serves images that have already been pulled by other nodes we want to trigger such an event to occur. We do this by creating pods using an image on two different nodes. When both pods have been created we can be assured that if Spegel is working properly the node that the second pod runs on will pull from the first node.

```shell
UPSTREAM_POD_NAME=$(kubectl --namespace spegel -l app.kubernetes.io/name=spegel get pods -o custom-columns=:metadata.name --no-headers | shuf -n 1)
UPSTREAM_NODE_NAME=$(kubectl --namespace spegel get pod ${UPSTREAM_POD_NAME} -o jsonpath="{.spec.nodeName}")
kubectl --namespace default run upstream --image=ubuntu:25.04 --restart=Never --overrides="{\"spec\":{\"nodeName\":\"${UPSTREAM_NODE_NAME}\",\"containers\":[{\"name\":\"ubuntu\",\"image\":\"ubuntu:25.04\",\"imagePullPolicy\":\"Always\",\"command\":[\"true\"]}]}}"
MIRROR_POD_NAME=$(kubectl --namespace spegel -l app.kubernetes.io/name=spegel get pods -o custom-columns=:metadata.name --no-headers | grep -v "^${UPSTREAM_POD_NAME}$" | shuf -n 1)
MIRROR_NODE_NAME=$(kubectl --namespace spegel get pod ${MIRROR_POD_NAME} -o jsonpath="{.spec.nodeName}")
kubectl --namespace default run mirror --image=ubuntu:25.04 --restart=Never --overrides="{\"spec\":{\"nodeName\":\"${MIRROR_NODE_NAME}\",\"containers\":[{\"name\":\"ubuntu\",\"image\":\"ubuntu:25.04\",\"imagePullPolicy\":\"Always\",\"command\":[\"true\"]}]}}"
kubectl --namespace default delete pod upstream mirror
```

Spegel has a debug web page that shows instance level stats. To access the page we need to port forward to the second Spegel pod.

```shell
kubectl --namespace spegel port-forward ${MIRROR_POD_NAME} 9090
```

While the port forward is running visit [http://localhost:9090/debug/web](http://localhost:9090/debug/web) to view debug page for the Spegel instance. If everything is working properly the `Last Mirror Success` box should show a duration. If the value is `Pending` it means that the image pull that we triggered was never served through Spegel, and that some configuration is incorrect.
