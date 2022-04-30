---
title: "概念"
linkTitle: "概念"
weight: 4
description: >
  了解 Merbridge 项目中一些关键的概念
---

## eBPF

eBPF 全称为 Extended Berkeley Packet Filter，顾名思义，这是一个用来过滤网络数据包的模块。例如 eBPF 的 sockops 和 redir 能力，就可以高效地过滤和拦截数据包。  

eBPF 是一项起源于 Linux 内核的革命性技术，可以在操作系统的内核中运行沙盒程序，能够安全、有效地扩展 Linux 内核的功能，无需改变内核的源代码，也无需加载内核模块。

## iptables

iptables 是建立在 netfilter 之上的流量过滤器，通过向 netfilter 的挂载点上注册钩子函数来实现对流量过滤和拦截。从 iptables 这个名字上可以看出有表的概念，iptables 通过把这些规则表挂载在 netfilter 的不同链上，对进出内核协议栈的流量数据包进行过滤或者修改。  

iptables 默认有 4 个表：

- Filter 表（数据过滤表）
- NAT 表（地址转换表）
- Raw 表（状态跟踪表）
- Mangle 表（包标记表）

iptables 默认有 5 个链：

- INPUT 链（入站规则）
- OUTPUT 链（出站规则）
- FORWARD 链（转发规则）
- PREROUTING 链（路由前规则）
- POSTROUTING 链（路由后规则）

## Service Mesh

中文名为服务网格，这是一个可配置的低延迟基础设施层，通过 API 接口处理应用服务之间的网络进程间通信。服务网格能确保容器化应用基础结构服务之间的通信快速、可靠和安全。网格提供的关键功能包括服务发现、负载均衡、安全加密和身份验证、故障恢复、可观测性等。  

服务网格通常会为每个服务实例注入一个 Sidcar 的代理实例。这些 Sidcar 会处理服务间的通信、监控和安全等问题。这样，开发人员就可以专注于服务中应用代码的开发、支持和维护，而运维团队负责服务网格以及应用的维护工作。  

目前最著名的服务网格架构是 Istio。

## Istio

Istio 是最初由 IBM、Google 和 Lyft 开源的服务网格技术。它可以透明地分层到分布式应用上，并提供服务网格的所有优点，例如流量治理、安全性和可观测性等。  

Istio 能够适配本地部署、云托管、Kubernetes 容器以及虚拟机上运行的服务程序。通常与 Kubernetes 平台上部署的微服务一起使用。  

从根本上讲，Istio 的工作原理是以 Sidcar 的形式将 Envoy 的扩展版本作为代理布署到每个微服务中。其使用的代理网络构成了 Istio 的数据平面。而这些代理的配置和管理在控制平面完成，为数据平面中的 Envoy 代理提供发现、配置和证书管理。

## Linkerd

Linkerd 是市场上出现的第一个服务网格。  

Linkerd 是 Buoyant 为 Kubernetes 设计的开源、超轻量级的服务网格。用 Rust 语言完全重写，使其尽可能小、轻和安全，它提供了运行时调试、可观测性、可靠性和安全性，而无需在分布式应用中更改代码。  

Linkerd 有三个基本组件：UI、数据平面和控制平面。Linkerd 通过在每个服务实例旁安装一组超轻、透明的代理来工作，这些代理会自动处理进出服务的所有流量。