---
title: "Merbridge 和 Cilium 有什么关系和区别？"
linkTitle: "Merbridge 和 Cilium 有什么关系和区别？"
date: 2022-04-23
weight: 1
---

# Merbridge 与 Cilium 有什么关系和区别

[Cilium](https://cilium.io/) 是一款基于 eBPF 为云原生应用提供诸多网络能力的优秀开源软件。作为 eBPF 领域的一个先行者，它有很多优秀的设计理念，值得我们学习。

Merbridge 同样基于 eBPF 技术，为服务网格应用提供更快速、更高效的连接能力，虽与 Cilium 的应用场景有所不同，但又是相互依存的关系。

Cilium 主要是为云原生应用提供一种基础的网络能力，如 Service 的 ClusterIP、NodePort、网络策略等。而 Merbridge 主要场景是使用 eBPF 技术代替 iptables，实现在服务网格场景下的流量拦截和转发，其本身还需要依赖底层 CNI（如 Cilium）的能力。

所以，Merbridge 仅专注于增强服务网格领域的网络能力，需要依赖类似于 Cilium 这样的底层 CNI，但是可以给服务网格领域的用户提供另外一个更好的选择。

## 感谢

Cilium 在 eBPF 领域耕耘多年，运营得非常成功，用各种方式向开源世界推广 eBPF 技术，输出了很多优秀的 Blog 和 Workshop 等。这也为 Merbridge 的应运而生提供了很多帮助和灵感。

我们开发团队从 Cilium 提供的资料中学习了很多关于 eBPF 的理论知识、实践方法和测试方法等，也与 Cilium 技术团队多有交流，也就是因为这些经历，才能让 Merbridge 这样的精巧项目孵化出来。

再次衷心感谢 Cilium 社区，非常欢迎大家来[社区](https://github.com/merbridge)一起讨论技术问题，推进 eBPF 相关技术发展壮大。相信随着 eBPF 技术的演进，Cilium 和 Merbridge 都会获益良多，携手共进。
