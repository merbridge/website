---
title: "Merbridge 和 Cilium 有什么关系和区别？"
linkTitle: "Merbridge 和 Cilium 有什么关系和区别？"
date: 2022-04-23
weight: 1
---

# Merbridge 和 Cilium 有什么关系和区别？

[Cilium](https://cilium.io/) 是一个优秀的基于 eBPF 的为云原生应用提供各种网络能力的开源软件。作为 eBPF 领域的一个优秀软件，有很多优秀的设计，值得我们学习。

Merbridge 同样基于 eBPF 技术，为服务网格应用提供更加快速和高效的连接能力，这与 Cilium 的应用场景是不同的，主要是以下原因：

Cilium 主要是为云原生应用提供网络能力，能够提供一种更加基础的网络能力，如 Service 的 ClusterIP、NodePort、网络策略 等。而 Merbridge 主要场景是为了使用 eBPF 技术代替 iptables，实现在服务网格场景下的流量拦截和转发，其本身还需要依赖底层 CNI（如 Cilium）的能力。
所以，Merbridge 仅仅是专注于服务网格领域的网络能力，需要依赖类似于 Cilium 这样的底层 CNI，但是可以给服务网格领域的用户另外的一个选择。

## 感谢

当然，作为 eBPF 领域非常早期且非常成功的软件，Cilium 在用各种方式向世界推广 eBPF 技术，输出了很多优秀的 Blog 和 Workshop 等。这也为 Merbridge 的出现提供了很多帮助。

我们从这些资料中学习了很多关于 eBPF 的理论知识、实践方法或者测试方法等，这都是很宝贵的资料，也帮助 Merbridge 能够快速的孵化出来。

感谢 Cilium 社区，我们也非常欢迎大家一起讨论各种相关技术。
