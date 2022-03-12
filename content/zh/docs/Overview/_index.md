---
title: "概览"
linkTitle: "概览"
weight: 1
description: >
  帮助您大致了解 Merbridge。
---

{{% pageinfo %}}
此处帮助您大致了解 Merbridge 项目。
{{% /pageinfo %}}

## Merbridge 是什么

Merbridge 是一个专门为服务网格设计，用于代替传统的 iptables 流量劫持技术，让服务网格的流量拦截和转发能力更加高效。

## 为什么需要 Merbridge？

在服务网格的场景中，为了能够做到在应用完全无感知的情况下，利用边车来进行流量治理，就需要把 Pod 的出入口流量都转发到边车进行治理。
在这种情况下，最通用的技术就是使用 iptables(netfilter) 的 REDIRECT 能力，将原本的流量转向边车，从而实现流量治理和其它相应功能。

在这种情况下，比如在流量入口方向，原本直接流向应用的流量，需要先经过 iptables(netfilter) 的转发，到边车之后，又要边车将流量转发到实际的应用。
在这个过程中，netfilter 的性能表现不是很好。而 eBPF 技术由于其独特的特性，也能够做到类似的事情，所以 Merbridge 就诞生了。

我们希望借助 eBPF 技术，来代替 iptables(netfilter)，让服务网格的效率更高。

## 在什么场景下需要 Merbridge？

TODO