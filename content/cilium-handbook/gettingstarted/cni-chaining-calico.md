::: {.only}
not (epub or latex or html)

WARNING: You are looking at unreleased Cilium documentation. Please use
the official rendered version released here: <https://docs.cilium.io>
:::

Calico
======

This guide instructs how to install Cilium in chaining configuration on
top of [Calico](https://github.com/projectcalico/calico).

Create a CNI configuration
--------------------------

Create a `chaining.yaml` file based on the following template to specify
the desired CNI chaining configuration:

``` {.yaml}
apiVersion: v1
kind: ConfigMap
metadata:
  name: cni-configuration
  namespace: kube-system
data:
  cni-config: |-
    {
      "name": "generic-veth",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "calico",
          "log_level": "info",
          "datastore_type": "kubernetes",
          "mtu": 1440,
          "ipam": {
              "type": "calico-ipam"
          },
          "policy": {
              "type": "k8s"
          },
          "kubernetes": {
              "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
          }
        },
        {
          "type": "portmap",
          "snat": true,
          "capabilities": {"portMappings": true}
        },
        {
          "type": "cilium-cni"
        }
      ]
    }
```

Deploy the `ConfigMap`{.interpreted-text role="term"}:

``` {.shell-session}
kubectl apply -f chaining.yaml
```

Deploy Cilium with the portmap plugin enabled
---------------------------------------------

Deploy Cilium release via Helm:

::: {.parsed-literal}

helm install cilium \\

:   \--namespace=kube-system \\ \--set cni.chainingMode=generic-veth \\
    \--set cni.customConf=true \\ \--set cni.configMap=cni-configuration
    \\ \--set tunnel=disabled \\ \--set enableIPv4Masquerade=false \\
    \--set enableIdentityMark=false
:::

::: {.note}
::: {.title}
Note
:::

The new CNI chaining configuration will *not* apply to any pod that is
already running the cluster. Existing pods will be reachable and Cilium
will load-balance to them but policy enforcement will not apply to them
and load-balancing is not performed for traffic originating from
existing pods.

You must restart these pods in order to invoke the chaining
configuration on them.
:::
