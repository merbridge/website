
---
title: "一行代码，使用 eBPF 代替 iptables 加速服务网格"
linkTitle: "一行代码，使用 eBPF 代替 iptables 加速服务网格"
date: 2022-03-01
weight: 1
---

# 一行代码使用 eBPF 代替 iptables 加速 Istio

## 介绍

以 Istio 为首的服务网格技术正在被越来越多的企业关注，其使用 Sidecar 借助 iptables 技术实现流量拦截，可以处理所有应用的出入口流量，以实现诸如治理、观测、加密等能力。

但是使用 iptables 的方式进行拦截，由于需要对出入口都拦截，会让原本主需要在内核态处理两次的链路变成四次，会损失不少性能，这在一些要求高性能的场景下显然是有影响的。

近两年，由于 eBPF 技术的兴起，不少围绕 eBPF 的项目也应声而出，eBPF 在可观测性和网络包的处理方面也有不少优秀的案例。如 Cilium、px.dev 等项目。

借助 eBPF 的 sockops 和 redir 能力，可以高效的处理数据包，再结合实际场景，那么我们就可以使用 eBPF 去代替 iptables 为 Istio 进行加速。

现在，我们开源了 Merbridge 项目，只需要在您的 Istio 集群执行以下命令，即可直接使用 eBPF 代替 iptables 实现网络加速！

```bash
kubectl apply -f https://raw.githubusercontent.com/merbridge/merbridge/main/deploy/all-in-one.yaml
```

> 注意：当前仅支持在 5.7 版本及以上的内核下运行，请事先升级您的内核版本。
> 

### eBPF 的 sockops 加速

网络连接本质上是 socket 的通讯，eBPF 提供了一个 [bpf_msg_redirect_hash]([https://man7.org/linux/man-pages/man7/bpf-helpers.7.html](https://man7.org/linux/man-pages/man7/bpf-helpers.7.html)) 函数，用来将应用发出的包，直接转发到对端的 socket 上面，可以极大的加速包在内核中的处理流程。

这里需要一个 sock_map，需要根据当前的数据包信息，从 sock_map 中挑选一个存在的 socket 连接，转发请求，所以，需要在 sockops 的 hook 处或者其它地方将 socket 信息保存到 sock_map，并提供根据 key 查到 socket 的规则（一般为四元组）。

## 原理

下面，将按照实际的场景，逐步的介绍 Merbridge 详细的设计和实现原理，这将让你对 Merbridge 或者 eBPF 有一个初步的了解。

### Istio 基于 iptables 的原理

![Istio 基于 iptables 的流量拦截原理](./imgs/1.png)

如上图所示，当外部流量相应访问应用的端口时，会在 iptables 中被 PREROUTING 拦截，最后转发到 Sidecar 容器的 15006 端口，然后交给 Envoy 来进行处理。（图中红色 1 2 3 4 的路径）

Envoy 根据从控制平面下发的规则进行处理，处理完成后，会发送请求给实际的容器端口。

当应用想要访问其它服务时，会在 iptables 中 OUTPUT 拦截，然后转发给 Sidecar 容器的 15001 端口（Envoy 监听）。（图中红色 9 10 11 12 的路径）然后和入口流量处理差不多。

由此可以看到，原本流量可以直接到应用端口，但是中间需要通过 iptables 转发到 Sidecar，然后又让 Sidecar 发送给应用，这无疑增加了开销。并且，iptables 的通用性决定了它的性能没有很理想。会在整条链路上增加不少延迟。

如果我们能使用 sockops 去直接连接 Sidecar 到应用的 Socket，这样可以使流量不经过 iptables，可以提高性能。

### 出口流量处理

如上所述，我们希望使用 eBPF 的 sockops 来绕过 iptables 以加速网络请求。同时，我们希望创造的是一个能够完全适配社区版 Istio，不做任何改造。所以，我们需要模拟 iptables 所做的操作。

这个时候我们在看回 iptables 本身，其使用 DNAT 功能做流量转发。

想要用 eBPF 模拟 iptables 的能力，那么就需要使用 eBPF 实现类似 iptables DNAT 的能力。

这里主要有两个点：

1. 修改连接发起时的目的地址，让流量能够发送到新的接口；
2. 让 Envoy 能识别原始的目的地址，以能够识别流量；

对于其中第一点，我们可以使用 eBPF 的 connect 程序来做，通过修改 `user_ip` 和 `user_port` 实现。

对于其中第二点，需要用到 ORIGINAL_DST 的概念。这个在内核中其实是在 netfilter 模块专属的。

其原理就是，应用程序（包括 Envoy）会在收到连接之后，调用 get_sockopts 函数，获取 ORIGINAL_DST，如果经过了 iptables 的 DNAT，那么 iptables 就会给当前的 socket 设置这个值，并把原有的 IP + 端口写入这个值，应用程序就可以根据连接拿到原有的目的地址。

那么我们就需要通过 eBPF 的 `get_sockopt` 程序来修改这个调用。（不用 **`bpf`_setsockopt`** 的原因是因为目前这个参数并不支持 `SO_ORIGINAL_DST` 的 optname）

参见下图，在应用向外发起请求时，会经过如下阶段：

1. 在应用向外发起连接时，connect 程序会将目标地址修改为 `127.x.y.z:15001`，并用 `cookie_original_dst` 保存原始目的地址。
2. 在 sockops 程序中，将当前 sock 和四元组保存在 `sock_pair_map` 中。同时，将四元组信息和对应的原始目的地址写入 `pair_original_dst` 中（之所以不用 cookie，是因为在 get_sockopt 程序中无法获取当前 cookie）。
3. Envoy 收到连接之后会调用 getsockopt 获取当前连接的目的地址，get_sockopt 程序会根据四元组信息从 `pair_original_dst` 取出原始目的地址并返回，由此连接完全建立。
4. 在发送数据阶段，redir 程序会根据四元组信息，从 `sock_pair_map` 中读取 sock，然后通过 `bpf_msg_redirect_hash` 进行直`接转发，加速请求。

![出口流量处理](./imgs/2.png)

其中，之所以在 connect 的时候，修改目的地址为 127.x.y.z 而不是 127.0.0.1，是因为在不同的 Pod 中，可能产生冲突的四元组，使用此方式即可巧妙的避开。（每个 Pod 间的目的 IP 就已经不同了，不存在冲突的情况）

### 入口流量处理

入口流量处理基本和出口流量类似，唯一差别：只需要将目的地址的端口改成 15006 即可。

但是，需要注意的是，由于 eBPF 不像 iptables 能在指定命名空间生效，它是全局的，这就造成如果我们将一个本来不是 Istio 所管理的 Pod，或者就是一个外部的 IP 地址，也做了这个操作的话，那就会引起严重问题，会请求直接无法建立连接。

所以这里我们设计了一个小的控制平面（以 DaemonSet 方式部署），其通过 Watch 所有的 Pod，类似于像 kubelet 那样获取当前节点的 Pod 列表，将已经被注入了 Sidecar 的 Pod IP 地址写入 `local_pod_ips` 这个 map。

当我们在做入口流量处理的时候，如果目的地址不在这个列表之中，我们就不做处理，让它走原来的逻辑，这样就可以比较灵活且简单的处理入口流量。

其他地，流程和出口流量流程一样。

![入口流量处理](./imgs/3.png)

### 同节点加速

通过入口流量处理，理论上，我们已经可以直接加速同节点的 Envoy 到 Envoy 的加速。但是存在一个问题。就是在这种场景下，Envoy 访问当前 Pod 的应用的时候会出错。

在 Istio 中，Envoy 访问应用的方式是使用当前 PodIP 加服务端口。经过上面入口流量处理章节，其实我们会发现，由于 PodIP 肯定也存在于 `local_pod_ips` 中，那么这个请求会被转发到 PodIP + 15006 端口，这显然是不行的，会造成无限递归。

那么我们也没办法在 eBPF 中获取当前 ns 的 IP 地址信息，怎么办？

为此，我们设计了一套反馈机制：

即，在 Envoy 尝试建立连接的时候，我们还是会走重定向到 15006 端口，但是，在 sockops 阶段，我们会判断源 IP 和目的地址 IP是否一致，如果一致，代表发送了错误的请求，那么我们会在 sockops 丢弃这个连接，并将当前的 ProcessID 和 IP 地址信息写入 `process_ip` 这个 map，让 eBPF 支持进程和 IP 的对应关系。

当下次请求发送时，我们直接从 `process_ip` 表检查目的地址是否和当前 IP 地址

> Envoy 会在请求失败的时候重试，且这个错误只会发生一次，后续的连接会非常快。
> 

![同节点加速](./imgs/4.png)

### 连接关系

在没有使用 Merbridge（eBPF） 优化之前，Pod 到 Pod 间的访问入下图所示：

![iptable 路径](./imgs/5.png)

在使用 Merbridge（eBPF）优化之后，出入口流量会使用直接跳过很多内核模块，提高性能：

![eBPF 路径](./imgs/6.png)

同时，如果两个 Pod 在同一台机器上，那么他们之间的通讯将更加高效：

![同节点 eBPF 路径](./imgs/7.png)

以上，通过使用 eBPF 在主机上对相应的连接进行处理，可以大幅度的减少内核处理流量的流程，提升服务之间的通讯质量。

## 加速效果

> 下面的测试只是一个基本的测试，不是非常严谨。
> 

下图展示了使用 eBPF 代替 iptables 之后，整体延迟的情况（越低越好）：

![延迟与连接数](./imgs/8.png)

下图展示了使用 eBPF 代替 iptables 之后，整体 QPS 的情况（越高越好）：

![延迟与 QPS](./imgs/9.png)

> 以上数据使用 wrk 测试得出。
> 

## Merbridge 项目

以上介绍的都是 Merbridge 项目的核心能力，其通过使用 eBPF 代替 iptables，可以在服务网格场景下，完全无感知的对流量通路进行加速。同时，我们不会对现有的 Istio 做任何修改，原有的逻辑依然畅通，这意味着，如果不再希望使用 eBPF，那么可以直接删除掉 DaemonSet，改为传统的 iptables 方式也不会出任何问题。

Merbridge 是一个完全独立的开源项目，此时还处于早期阶段，我们希望可以有更多的用户或者开发者参与其中，使用先进的技术能力，优化我们的服务网格。

项目地址：[https://github.com/merbridge/merbridge](https://github.com/merbridge/merbridge)

参考文档：

* [https://ebpf.io/](https://ebpf.io/)

* [https://cilium.io/](https://cilium.io/)
* [Merbridge on GitHub](https://github.com/merbridge/merbridge)
* [Using eBPF instead of iptables to optimize the performance of service grid data plane](https://developpaper.com/kubecon-2021-%EF%BD%9C-using-ebpf-instead-of-iptables-to-optimize-the-performance-of-service-grid-data-plane/) by Liu Xu, Tencent
* [Sidecar injection and transparent traffic hijacking process in Istio explained in detail](https://jimmysong.io/en/blog/sidecar-injection-iptables-and-traffic-routing/) by Jimmy Song, Tetrate
* [Accelerate the Istio data plane with eBPF](https://01.org/blogs/xuyizhou/2021/accelerate-istio-dataplane-ebpf-part-1) by Yizhou Xu, Intel
* [Envoy's Original Destination filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/listener_filters/original_dst_filter)
