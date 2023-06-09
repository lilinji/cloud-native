---
weight: 1
title: 路由
date: '2022-06-17T12:00:00+08:00'
type: book
---

## 封装 

当没有提供配置时，Cilium 会自动在此模式下运行，因为它是对底层网络基础设施要求最低的模式。

在这种模式下，所有集群节点使用基于 UDP 的封装协议  [`VXLAN`](https://docs.cilium.io/en/stable/glossary/#term-vxlan) 或  [`Geneve`](https://docs.cilium.io/en/stable/glossary/#term-geneve)。Cilium 节点之间的所有流量都被封装。

### 对网络的要求 

- 封装依赖于正常的节点到节点的连接。这意味着如果 Cilium 节点已经可以互相到达，那么所有的路由要求都已经满足了。

- 底层网络和防火墙必须允许封装数据包：

  | 封装方式      | 端口范围 / 协议 |
  | ------------- | --------------- |
  | VXLAN（默认） | 8472/UDP        |
  | Geneve        | 6081/UDP        |

### 封装模式的优点

封装模式具有以下优点：

- **简单**

  连接集群节点的网络不需要知道 PodCIDR。集群节点可以产生多个路由或链路层域。只要集群节点可以使用 IP/UDP 相互访问，底层网络的拓扑就无关紧要。

- **寻址空间**

  由于不依赖于任何底层网络限制，如果相应地配置了 PodCIDR 大小，可用的寻址空间可能会更大，并且允许每个节点运行任意数量的 Pod。

- **自动配置**

  当与 Kubernetes 等编排系统一起运行时，集群中所有节点的列表（包括它们关联的分配前缀节点）会自动提供给每个代理。加入集群的新节点将自动合并到网格中。

- **身份上下文**

  封装协议允许将元数据与网络数据包一起携带。Cilium 利用这种能力来传输元数据，例如源安全身份。身份传输是一种优化，旨在避免在远程节点上进行一次身份查找。

### 封装模式的缺点

封装模式有以下缺点：

- **MTU 开销**

  由于添加了封装标头，可用于有效负载的有效 MTU 低于本地路由（VXLAN 的每个网络数据包 50 字节）。这会导致特定网络连接的最大吞吐率较低。这可以通过启用巨型帧（每 1500 字节 50 字节的开销对比每 9000 字节的 50 字节开销）在很大程度上得到缓解。

## 本地路由

本地路由数据路径使用 `tunnel: disabled` 启用并启用本机数据包转发模式。本地数据包转发模式利用 Cilium 运行的网络的路由功能，而不是执行封装。

![本地路由模式](../native_routing.png "本地路由模式")

在本地路由模式下，Cilium 会将所有未发送到另一个本地端点的数据包委托给 Linux 内核的路由子系统。这意味着数据包将被路由，就好像本地进程会发出数据包一样。因此，连接集群节点的网络必须能够路由 PodCIDR。

当配置本地路由时，Cilium 会在 Linux 内核中自动启用 IP 转发。

### 对网络的要求

为了运行本地路由模式，连接运行 Cilium 的主机的网络必须能够使用分配给 pod 或其他工作负载的地址转发 IP 流量。

节点上的 Linux 内核必须知道如何转发运行 Cilium 的所有节点的 pod 或其他工作负载的数据包。这可以通过两种方式实现：

1. 节点本身不知道如何路由所有 pod IP，但网络上存在一个知道如何到达所有其他 pod 的路由器。在这种情况下，Linux 节点配置为包含指向此类路由器的默认路由。该模型用于云提供商网络集成。
2. 每个单独的节点都知道所有其他节点的所有 Pod IP，并且路由被插入到 Linux 内核路由表中来表示这一点。如果所有节点共享一个 L2 网络，则可以通过启用 `auto-direct-node-routes: true` 选项来解决此问题。否则，必须运行额外的系统组件（例如 BGP 守护程序）来分发路由。请参阅 [使用 kube-router 运行 BGP](https://docs.cilium.io/en/stable/gettingstarted/kube-router/#kube-router) 指南，了解如何使用 kube-router 项目实现此目的。

### 配置

必须设置以下配置选项以在本地路由模式下运行数据路径：

- `tunnel: disabled`：启用本机路由模式。
- `ipv4-native-routing-cidr: x.x.x.x/y`：设置可以执行本机路由的 CIDR。
