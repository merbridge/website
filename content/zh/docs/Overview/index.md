---
title: "概览"
linkTitle: "概览"
weight: 1
description: >
  本文概述了 Merbridge 的含义、特性、适用场景等内容。
---

## Merbridge 是什么

Merbridge 专为服务网格设计，使用 eBPF 代替传统的 iptables 劫持流量，能够让服务网格的流量拦截和转发能力更加高效。

eBPF (extended Berkeley Packet Filter) 技术可以在 Linux 内核中运行用户编写的程序，而且不需要修改内核代码或加载内核模块，目前广泛应用于网络、安全、监控等领域。相比传统的 iptables 流量劫持技术，基于 eBPF 的 Merbridge 可以绕过很多内核模块，缩短边车和服务间的数据路径，从而加速网络。Merbridge 没有对现有的 Istio 作出任何修改，原有的逻辑依然畅通。这意味着，如果您不想继续使用 eBPF，直接删除相关的 DaemonSet 就能恢复为传统的 iptables 方式，不会出现任何问题。

## Merbridge 有哪些特性

Merbridge 的核心特性包括：

- 出口流量处理

  Merbridge 使用 eBPF 的 `connect` 程序，修改 `user_ip` 和 `user_port` 以改变连接发起时的目的地址，从而让流量能够发送到新的接口。为了让 Envoy 识别出原始目的地址，应用程序（包括 Envoy）会在收到连接之后调用 `get_sockopts` 函数，获取 ORIGINAL_DST。

- 入口流量处理

  入口流量处理与出口流量处理基本类似。需要注意的是，eBPF 是全局性的，不能在指定的命名空间生效。因此，如果对原本不是由 Istio 管理的 Pod 或者对外部的 IP 地址执行此操作，就会导致请求无法建立连接。为了解决此问题，Merbridge 设计了一个小的控制平面（以 DaemonSet 方式部署），通过 Watch 所有的 Pod，用类似于 kubelet 的方式获取当前节点的 Pod 列表，然后将已经注入 Sidecar 的 Pod IP 地址写入 `local_pod_ips map`。如果流量的目的地址不在该列表中，Merbridge 就不做处理，转而使用原来的逻辑。这样就可以灵活且便捷地处理入口流量。

- 同节点加速

  在 Istio 中，Envoy 使用当前 PodIP 加服务端口来访问应用程序。由于 PodIP 肯定也存在于 `local_pod_ips` ，所以请求就会被转发到 PodIP + 15006 端口。这样会造成无限递归，不能在 eBPF 获取当前命名空间的 IP 地址信息。因此，需要一套反馈机制：在 Envoy 尝试建立连接时仍然重定向到 15006 端口，在 sockops 阶段判断源 IP 和目的 IP 是否一致。如果一致，说明发送了错误的请求，需要在 sockops 丢弃该连接，并将当前的 ProcessID 和 IP 地址信息写入 `process_ip map`，让 eBPF 支持进程和 IP 的对应关系。下次发送请求时直接从 `process_ip` 表检查目的地址是否与当前 IP 地址一致。Envoy 会在请求失败时重试，且这个错误只会发生一次，后续的连接会非常快。

## 为什么需要 Merbridge

在服务网格场景中，为了能在应用程序完全无感知的情况下利用边车进行流量治理，需要把 Pod 的出入口流量都转发到边车。在这种情况下，最常见的解决方案就是使用 iptables (netfilter) 的重定向能力。这种方案的缺点是增加了网络延迟，因为 iptables 对出口流量和入口流量都进行拦截。以入口流量为例，原本直接流向应用的流量，需要先由 iptables 转发到边车，再由边车将流量转发到实际的应用。原本只需要在内核态处理两次的链路如今变成四次，损失了不少性能。

幸运的是，eBPF 技术提供了一个 `bpf_msg_redirect_hash` 函数，可以将应用发出的数据包直接转发到对端的 socket，从而极大地加速数据包在内核中的处理流程。我们希望用 eBPF 技术代替 iptables 提高服务网格的效率，于是 诞生了 Merbridge。

## Merbridge 适用哪些场景

如果您遇到下列任一问题，建议使用 Merbridge：

1. 在需要高性能连接的场景下，使用 iptables 会增加延迟。
    - 由于集群中容器数量增加，iptables 控制面和数据面的性能会急剧下降。在 iptables 控制面的接口设计中，每添加一条规则都需要遍历并修改所有的规则。
    - 由于 Pod 生命周期越来越短，有时甚至只有几秒钟，这就需要快速更新 iptables 规则，使用 IP 地址进行安全过滤的系统将承受越来越大的压力。
    - 由于使用 iptables 实现透明劫持需要借助 conntrack 模块跟踪连接，连接较多时会造成大量消耗。
2. 系统因某些原因不能使用 iptables。
   - 有时需同时处理大量活动连接，但使用 iptables 容易出现 conntrack 表满的情况。
   - 有时又需每秒处理极大数量的连接，但超出了 conntrack 表的限制。例如，在超时设置为 120 秒且表容量是 128k 的情况下，如果尝试每秒处理 1100 个连接，就会超出 conntrack 表的限制（128k/120秒 = 1092 连接/秒）。
3. 出于安全考虑不能为普通的 Pod 授予太多权限，但使用 Istio（若无 CNI）必须允许 Pod 获得更多权限。
   - 运行 init 容器，可能需要 `NET_ADMIN` 等权限。
   - 运行 iptables 命令，对应的进程可能需要 `CAP_NET_ADMIN` 权限。
   - 挂载文件系统，对应的进程可能需要 `CAP_SYS_ADMIN` 权限。

## Merbridge 如何改变连接关系

使用 eBPF 在主机上处理连接，可以显著简化内核处理流量的流程，提升服务之间的通讯质量。

- 如果不用 Merbridge (eBPF)，Pod 到 Pod 间的访问连接关系如下图。

![iptable 路径](./imgs/iptables_path.png)
> 图片参考：[Accelerating Envoy and Istio with Cilium and the Linux Kernel](https://pt.slideshare.net/ThomasGraf5/accelerating-envoy-and-istio-with-cilium-and-the-linux-kernel/22)

- 使用 Merbridge (eBPF) 优化之后，处理出入口流量时会跳过很多内核模块，从而加速网络。

![eBPF 路径](./imgs/eBPF_path.png)
> 图片参考：[Accelerating Envoy and Istio with Cilium and the Linux Kernel](https://pt.slideshare.net/ThomasGraf5/accelerating-envoy-and-istio-with-cilium-and-the-linux-kernel/22)

- 如果两个 Pod 在同一节点上，使用 Merbridge (eBPF) 能让 Pod 之间的通讯更加高效。

![同节点 eBPF 路径](./imgs/sameNode_eBPF_path.png)
> 图片参考：[Accelerating Envoy and Istio with Cilium and the Linux Kernel](https://pt.slideshare.net/ThomasGraf5/accelerating-envoy-and-istio-with-cilium-and-the-linux-kernel/22)

Merbridge 是完全独立的开源项目，目前仍处于早期阶段。希望有更多的用户或开发者参与其中，不断完善 Merbridge，共同优化服务网格。如果您发现了 Merbridge 的漏洞而且有兴趣帮助修复，非常欢迎您[提交 Pull Request](https://github.com/merbridge/merbridge/pulls)，附上您的修复代码，我们会及时处理您的 PR。