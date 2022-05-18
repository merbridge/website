---
title: "Merbridge CNI 模式"
linkTitle: "Merbridge CNI 模式"
date: 2022-05-18
weight: 1
description: 此篇博客将向您介绍 Merbridge CNI 的工作原理。
---

Merbridge CNI 模式的出现，旨在能够更好地适配服务网格的功能。之前没有 CNI 模式时，Merbridge 能够做得事情比较有限。其中最大的问题是不能适配注入 Istio 的 Sidecar Annotation，这就导致 Merbridge 无法排除某些端口或 IP 段的流量等。
同时，由于之前 Merbridge 只处理 Pod 内部的连接请求，这就导致，如果是外部发送到 Pod 的流量，Merbridge 将无法处理。

为此，我们精心设计了 Merbridge CNI，旨在解决这些问题。

## 为什么需要 CNI 模式？

其一，之前的 Merbridge 只有一个很小的控制面，其监听 Pod 资源，将当前节点的 IP 信息写入 `local_pod_ips` 的 map，以供 connect 使用。
但是，connect 程序由于工作在主机内核层，其无法知道当前正在处理的是哪个 Pod 的流量，就没法处理如 `excludeOutboundPorts` 等配置。
为了能够适配注入 `excludeOutboundPorts` 的 Sidecar Annotation，我们需要让 eBPF 程序能够得知当前正在处理哪个 Pod 的请求。

为此，我们设计了一套方法，与 CNI 配合，能够获取当前 Pod 的 IP，以适配针对 Pod 的特殊配置。

其二，在之前的 Merbridge 版本中，只有 connect 会处理主机发起的请求，这在同一台主机上的 Pod 互相通讯时，是没有问题的。但是在不同主机之间通讯时就会出现问题，因为按照之前的逻辑，在跨节点通讯时流量不会被修改，这会导致在接收端还是离不开 iptables。

这次，我们依靠 XDP 程序，解决入口流量处理的问题。因为 XDP 程序需要挂载网卡，所以也需要借助 CNI。

## CNI 如何解决问题？

这里我们将探讨 CNI 的工作原理，以及如何使用 CNI 来解决问题。

### 如何通过 CNI 让 eBPF 程序获取当前正在处理的 Pod IP？

我们通过 CNI，在 Pod 创建的时候，将 Pod 的 IP 信息写入一个 Map（`mark_pod_ips_map`），其 Key 为一个随机的值，Value 为 Pod 的 IP。然后，在当前 Pod 的 NetNS 里面监听一个特殊的端口 39807，将 Key 使用 `setsockopt` 写入这个端口 socket 的 mark。

在 eBPF 中，我们通过 `bpf_sk_lookup_tcp` 取得端口 39807 的 Mark 信息，然后从 `mark_pod_ips_map` 中即可取得当前 NetNS（也是当期 Pod）的 IP。

有了当前 Pod IP 之后，我们可以根据这个 Pod 的配置，确认流量处理路径（比如 `excludeOutboundPorts`）。

同时，我们还使用 Pod 优化了之前解决四元组冲突的方案，改为使用 `bpf_bind` 绑定源 IP，目的 IP 直接使用 `127.0.0.1`，为了后续支持 IPv6 做准备。

### 如何处理入口流量？

为了能够处理入口流量，我们引入了 XDP 程序，XDP 程序作用在网卡上，能够对原始数据包做修改。
我们借助 XDP 程序，在流量到达 Pod 的时候，修改目的端口为 15006 以完成流量转发。

同时考虑到可能存在主机直接访问 Pod 的情况，也为了减小影响范围，我们选择将 XDP 程序附加到 Pod 的网卡上。这就需要借助 CNI 的能力，在创建 Pod 时进行附加操作。

## 如何体验 CNI 模式？

CNI 模式默认被关闭，需要手动开启。

可以使用以下命令一键开启：

```bash
curl -sSL https://raw.githubusercontent.com/merbridge/merbridge/main/deploy/all-in-one.yaml | sed 's/--cni-mode=false/--cni-mode=true/g' | kubectl apply -f -
```

## 注意事项

### CNI 模式处于测试阶段

CNI 模式刚被设计和开发出来，可能存在不少问题，我们欢迎大家在测试阶段进行反馈，或者提出更好的建议，以帮助我们改进 Merbridge！

如果需要使用注入 Istio perf benchmark 等工具进行测试性能，请开启 CNI 模式，否则会导致性能测试结果不准确。

### 需要注意主机是否可开启 hardware-checksum 能力

为了保证 CNI 模式的正常运行，我们默认关闭了 hardware-checksum 能力，这可能会影响到网络性能。建议大家在开启 CNI 模式前，先确认主机是否可开启 hardware-checksum 能力。如果可以开启，建议设置 `--hardware-checksum=true` 以获得最佳的性能表现。

测试方法：`ethtool -k <网卡> | grep tx-checksum-ipv4` 为 on 表示开启。
