Requirements
============

Make sure your Kubernetes environment is meeting the requirements:

-   Kubernetes \>= 1.16
-   Linux kernel \>= 4.9
-   Kubernetes in CNI mode
-   Mounted eBPF filesystem mounted on all worker nodes
-   Recommended: Enable PodCIDR allocation (`--allocate-node-cidrs`) in
    the `kube-controller-manager` (recommended)

Refer to the section `k8s_requirements`{.interpreted-text role="ref"}
for detailed instruction on how to prepare your Kubernetes environment.
