---
title: Merbridge 支持 Ambient Mesh，无惧 CNI 兼容性！
linkTitle: Merbridge 支持 Ambient Mesh，无惧 CNI 兼容性！
date: 2022-11-11
weight: 1
description: 此篇博客介绍 Merbridge 支持 Ambient Mesh 的新特性。
author: Kebe Liu
---

在 [深入 Ambient Mesh - 流量路径](/zh/blog/2022/10/13/ambient-mesh-data-path/) 一文中，我们分析了 Ambient Mesh 如何实现在不引入 Sidecar 的情况下，将特定 Pod 的出入口流量转发到 ztunnel 进行处理。我们可以发现，其通过 iptables + TPROXY + 路由表的方式进行实现，路径比较长，理解起来比较复杂。而且，由于其使用 mark 做路由标记，这可能导致在某些同样依赖 CNI 的网络下无法正常工作，或者在一些 Bridge 模式的网络 CNI 下无法工作，会极大的限制 Ambient Mesh 使用场景。

Merbridge 主要场景就是使用 eBPF 代替 iptables 为服务网格应用加速。Ambient Mesh 作为 Istio 服务网格的全新模式，Merbridge 自然也要支持这种新模式。
iptables 技术强大，为很多软件实现了各种功能，但在实际应用中也存在一些缺陷。首先，iptables 使用线性匹配方式，当多个应用使用相同能力时可能会产生冲突，进而导致某些功能不可用。其次，虽然其足够灵活，但是仍旧无法实现像 eBPF 一样自由编程的能力。
所以，使用 eBPF 技术代替 iptables 的能力，帮助 Ambient Mesh 实现流量拦截，这应该是一项令人兴奋的技术。

## 确定目标

通过 [深入 Ambient Mesh - 流量路径](/zh/blog/2022/10/13/ambient-mesh-data-path/) 一文，我们可以知道的是，我们最终的目标有两个：

- 将位于 Ambient Mesh 模式下的 Pod 向外发出的流量，拦截到 ztunnel 的 15001 端口。
- 当主机程序向 Ambient Mesh 模式下的 Pod 发送流量时，应该将流量重定向到 ztunnel 的 15006 端口。

至于其中的 istioin 等网卡，完全是为了配合 Ambient Mesh 原有模式设计的，所以无需关注。

## 分析难点

在运行模式上，Ambient Mesh 模式和 Sidecar 模式存在很大区别，根据 Istio 官方的定义，将一个 Pod 加入 Ambient Mesh 并不需要重启 Pod，在 Pod 中也不存在 Sidecar 相关进程，这就意味着两个问题：

1. 之前 Merbridge 使用 CNI 的方案，让 eBPF 程序能够感知到当前 Pod 的 IP 地址从而做出策略判断的方式将失效，因为 Pod 加入或移除 Ambient Mesh 模式并不会重启，也不会调用 CNI 插件，需要重新考量。
2. 以前的流量拦截，只需要在 eBPF 的 connect 钩子中，将目标地址改为 `127.0.0.1:15001`，但是在 Ambient Mesh 模式下，需要将目标 IP 换成 ztunnel 的 IP 地址。

同时，由于 Ambient Mesh 模式下的 Pod 中，并不存在 Sidecar 相关进程，所以之前 Merbridge 用查看当前 Pod 中是否监听了 15006 等端口的方式也不再适用，需要重新设计方案判断进程的运行环境。

所以，基于上述分析，基本上需要重新设计整个拦截方案，以便让 Merbridge 支持 Ambient Mesh 模式。

总结下来，我们需要做这些事情：

- 重新设计判断 Pod 是否加入 Ambient Mesh 模式的方案
- 不依赖 CNI，实现 eBPF 感知当前 Pod IP 的方案
- 让 eBPF 程序知道当前节点的 ztunnel 的 IP 地址方案

## 解决方案

在  0.7.2 版本中，我们引入了使用 cgroup id 加速 connect 程序的性能，一般地，每一个 Pod 中的容器都有一个对应的 Cgroup id，我们可以在 bpf 程序中，通过 `bpf_get_current_cgroup_id` 函数获取到。通过将 IP 信息写入专用的 `cgroup_info_map` ，可以优化 connect 程序的运行速度。

因为我们不能像 CNI 模式下，在 Pod 加入 Ambient Mesh 模式下的时候，通过 CNI 在网络命名空间中监听一个特殊的端口，用于存储 Pod 的信息。所以，我们可以借助 cgroup id，如果能讲 cgroup id 与 Pod IP 产生关联，那就可以在 eBPF 中获取到当前 Pod IP 信息。

由于不能依赖 CNI，所以我们需要改变我们获取 Pod 状态变更信息的方案。为此，我们通过观测进程的创建销毁信息来探测本地 Pod 的创建和销毁等操作，创建了一个新的工具，用来观测主机上进程的变更信息：[process-watcher 项目](https://github.com/merbridge/process-watcher)。

通过从进程 ID 读取所在的 cgroup id 和 ip 信息，将 cgroup id 和 ip 等相关信息，写入 `cgroup_info_map` 

```go
tcg := cgroupInfo{
		ID:            cgroupInode,
		IsInMesh:      in,
		CgroupIp:      *(*[4]uint32)(_ip),
		Flags:         flag,
		DetectedFlags: cgrinfo.DetectedFlags | AMBIENT_MESH_MODE_FLAG | ZTUNNEL_FLAG,
	}
	return ebpfs.GetCgroupInfoMap().Update(&cgroupInode, &tcg, ebpf.UpdateAny)
```

然后，我们可以在 eBPF 中获取到当前 Cgroup 所关联的信息：

```c
__u64 cgroup_id = bpf_get_current_cgroup_id();
void *info = bpf_map_lookup_elem(&cgroup_info_map, &cgroup_id);
```

通过这里，我们可以知道，当前的容器是否启用了 Ambient Mesh 模式、是否在网格中等信息。

其次，对于 ztunnel 的 IP 地址，在 istio 中，通过增加网卡绑定固定 IP 的方式实现，这种方案存在可能冲突的风险，同时，也可能在某些情况下（比如 SNAT），会造成原地址信息丢失的情况。所以 Merbridge 直接放弃到此方案，直接在控制面获取 ztunnel 的 IP 地址，写入 map，让 eBPF 程序读取（这样速度更快）。

```c
static inline __u32 *get_ztunnel_ip()
{
    __u32 ztunnel_ip_key = ZTUNNEL_KEY;
    return (__u32 *)bpf_map_lookup_elem(&settings, &ztunnel_ip_key);
}
```

然后，可以利用 connect 程序，重写目标地址：

```c
ctx->user_ip4 = ztunnel_ip[3];
ctx->user_port = bpf_htons(OUT_REDIRECT_PORT);
```

通过与 Cgroup ID 的关联，我们即可以实现在 eBPF 中获取当前进程所在 Pod 的 IP 地址，从而进行策略操作，将输入 Ambient Mesh 模式下 Pod 发出的流量转发到 ztunnel，让其进行处理，从而实现 Merbridge 在 Ambient Mesh 模式下的兼容。

这将是对所有 CNI 都适配的一种能力，可以避免原有 Ambient Mesh 模式不能很好的在大多数 CNI 模式下无法工作的问题。

## 使用与反馈

由于目前 Ambient 任处于早期阶段，且 Merbridge 对于 Ambient 模式的支持也相对初级，还有一些问题没有得到很好的解决，所以 Ambient 模式还没有合并入主分支，如果想要体验 Merbridge 代替 iptables 为 Ambient Mesh 实现流量拦截的能力，可以按照如下的方式操作（首先需要安装好 Ambient Mesh 模式的网格）：

1. 禁用 Istio CNI（可以通过安装的时候设置 `--set components.cni.enabled=false` ，或者通过删除 Istio CNI 的 DaemonSet `kubectl -n istio-system delete ds istio-cni` 的方式）。
2. 删除 ztunnel 的 init 容器（它会初始化 iptables 规则、网卡等，这是不需要的）。
3. 使用 `kubectl apply -f https://github.com/merbridge/merbridge/raw/ambient/deploy/all-in-one.yaml` 安装支持 Merbridge 。

等待 Merbridge 就绪之后，即可以使用 Ambient Mesh 的所有能力。

***注意：**

1. 当前不支持在 Kind 下使用 Ambient 模式（将在后续支持）；
2. 主机内核版本需要大于等于 5.7；
3. 需要开启 Cgroupv2；
4. 此模式也兼容 Sidecar 模式；
5. Ambient 模式下安装默认会开启 debug 模式，会对性能造成一定影响。

更多实现细节可以查看[源码](https://github.com/merbridge/merbridge/tree/ambient)。

如果遇到任何问题，可在 [Slack](https://join.slack.com/t/merbridge/shared_invite/zt-11uc3z0w7-DMyv42eQ6s5YUxO5mZ5hwQ) 中与我们反馈，或加入我们的技术交流微信群。
