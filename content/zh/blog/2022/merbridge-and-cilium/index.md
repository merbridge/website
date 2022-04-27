---
title: "Merbridge 和 Cilium"
linkTitle: "Merbridge 和 Cilium"
date: 2022-04-23
weight: 1
---

# Merbridge 与 Cilium

[Cilium](https://cilium.io/) 是一款基于 eBPF 为云原生应用提供诸多网络能力的优秀开源软件，有很多很棒的设计。其中之一，Cilium 设计了一套基于 sockmap 的 redir 能力，帮助加速网络通讯，这给了我们很大的启发，也是 Merbridge 提供网络加速的基础，这真是一个非常棒的设计。

Merbridge 借助于 Cilium 为我们打下的好基础，加上我们在服务网格领域做地一些针对性的适配，让大家可以更加方便的将 eBPF 技术应用于服务网格。

我们开发团队从 Cilium 提供的资料中学习了很多关于 eBPF 的理论知识、实践方法和测试方法等，也与 Cilium 技术团队多有交流，也就是因为这些经历，才能有 Merbridge 项目的诞生。

再次衷心感谢 Cilium 项目和社区，以及 Cilium 的这些优秀的设计。
