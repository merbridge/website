---
title: 深入 Ambient Mesh - 流量路径
linkTitle: 深入 Ambient Mesh - 流量路径
date: 2022-10-13
weight: 1
description: 此篇博客介绍 Ambient Mesh 中数据面的流量路径。
author: Kebe Liu
---

Ambient Mesh 发布已经有一段时间，也有不少文章讲述了其用法和架构。本文将深入梳理数据面流量在 Ambient 模式下的路径，帮助大家全面地理解 Ambient 数据面的实现方案。

在阅读本文之前，请先阅读 [Ambient Mesh 介绍](https://istio.io/latest/blog/2022/introducing-ambient-mesh/) 了解 Ambient Mesh 的基本架构。

> 为了方便阅读和同步实践，本文使用的环境按照 [Ambient 使用](https://istio.io/latest/blog/2022/get-started-ambient/) 的方式进行部署。

## 从发起请求的一刻开始

为了探究流量路径，首先我们分析同在 Ambient 模式下的两个服务互相访问的情况（仅 L4 模式，不同节点）。

在 default 命名空间启用 Ambient 模式后，所有的服务都将具备网格治理的能力。

我们的分析从这条命令开始： `kubectl exec deploy/sleep -- curl -s http://productpage:9080/ | head -n1`

在 Sidecar 模式下，Istio 通过 iptables 进行流量拦截，当在 sleep 的 Pod 中执行 curl 时，流量会被 iptables 转发到 Sidecar 的 15001 端口进行处理。
但是在 Ambient 模式下，在 Pod 中不存在 Sidecar，且开启 Ambient 模式也不需要重启 Pod，那它的请求如何确保被 ztunnel 处理呢？

## 出口流量拦截

要了解出口流量拦截的方案，我们首先可以看一下控制面组件：

```console
kebe@pc $ kubectl -n istio-system get po
NAME                                   READY   STATUS    RESTARTS   AGE
istio-cni-node-5rh5z                   1/1     Running   0          20h
istio-cni-node-qsvsz                   1/1     Running   0          20h
istio-cni-node-wdffp                   1/1     Running   0          20h
istio-ingressgateway-5cfcb57bd-kx9hx   1/1     Running   0          20h
istiod-6b84499b75-ncmn7                1/1     Running   0          20h
ztunnel-nptf6                          1/1     Running   0          20h
ztunnel-vxv4b                          1/1     Running   0          20h
ztunnel-xkz4s                          1/1     Running   0          20h
```

在 Ambient 模式下 istio-cni 变成了默认组件。
而在 Sidecar 模式下，istio-cni 主要是为了避免使用 istio-init 容器处理 iptables 规则而造成权限泄露等情况推出的 CNI 插件。
但是在 Ambient 模式下，理论上不需要 Sidecar，为什么还需要 istio-cni 呢？

我们可以看一下日志：

```console
kebe@pc $ kubectl -n istio-system logs istio-cni-node-qsvsz
...
2022-10-12T07:34:33.224957Z	info	ambient	Adding route for reviews-v1-6494d87c7b-zrpks/default: [table 100 10.244.1.4/32 via 192.168.126.2 dev istioin src 10.244.1.1]
2022-10-12T07:34:33.226054Z	info	ambient	Adding pod 'reviews-v2-79857b95b-m4q2g/default' (0ff78312-3a13-4a02-b39d-644bfb91e861) to ipset
2022-10-12T07:34:33.228305Z	info	ambient	Adding route for reviews-v2-79857b95b-m4q2g/default: [table 100 10.244.1.5/32 via 192.168.126.2 dev istioin src 10.244.1.1]
2022-10-12T07:34:33.229967Z	info	ambient	Adding pod 'reviews-v3-75f494fccb-92nq5/default' (e41edf7c-a347-45cb-a144-97492faa77bf) to ipset
2022-10-12T07:34:33.232236Z	info	ambient	Adding route for reviews-v3-75f494fccb-92nq5/default: [table 100 10.244.1.6/32 via 192.168.126.2 dev istioin src 10.244.1.1]
```

我们可以看到，对于在 Ambient 模式下的 Pod，istio-cni 做了两件事情：

1. 添加 Pod 到 ipset
2. 添加了一个路由规则到 table 100（后面介绍用途）

我们可以在其所在的节点上查看一下 ipset 里面的内容（注意，这里使用 kind 集群，需要用 `docker exec` 先进入所在主机）：

```console
kebe@pc $ docker exec -it ambient-worker2 bash
root@ambient-worker2:/# ipset list
Name: ztunnel-pods-ips
Type: hash:ip
Revision: 0
Header: family inet hashsize 1024 maxelem 65536
Size in memory: 520
References: 1
Number of entries: 5
Members:
10.244.1.5
10.244.1.7
10.244.1.8
10.244.1.4
10.244.1.6
```

我们发现这个 Pod 所在的节点上有一个 ipset，其中保存了很多 IP。这些 IP 是 PodIP：

```console
kebe@pc $ kubectl get po -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP           NODE              NOMINATED NODE   READINESS GATES
details-v1-76778d6644-wn4d2       1/1     Running   0          20h   10.244.1.9   ambient-worker2   <none>           <none>
notsleep-6d6c8669b5-pngxg         1/1     Running   0          20h   10.244.2.5   ambient-worker    <none>           <none>
productpage-v1-7c548b785b-w9zl6   1/1     Running   0          20h   10.244.1.7   ambient-worker2   <none>           <none>
ratings-v1-85c74b6cb4-57m52       1/1     Running   0          20h   10.244.1.8   ambient-worker2   <none>           <none>
reviews-v1-6494d87c7b-zrpks       1/1     Running   0          20h   10.244.1.4   ambient-worker2   <none>           <none>
reviews-v2-79857b95b-m4q2g        1/1     Running   0          20h   10.244.1.5   ambient-worker2   <none>           <none>
reviews-v3-75f494fccb-92nq5       1/1     Running   0          20h   10.244.1.6   ambient-worker2   <none>           <none>
sleep-7b85956664-z6qh7            1/1     Running   0          20h   10.244.2.4   ambient-worker    <none>           <none>
```

所以，这个 ipset 保存了当前节点上所有处于 Ambient 模式下的 PodIP 列表。

那这个 ipset 在哪可以用到呢？

我们看一下 iptables 规则，可以发现：

```console
root@ambient-worker2:/# iptables-save
*mangle
...
-A POSTROUTING -j ztunnel-POSTROUTING
...
-A ztunnel-PREROUTING -p tcp -m set --match-set ztunnel-pods-ips src -j MARK --set-xmark 0x100/0x100
```

通过这个我们知道，当节点上处于 Ambient 模式的 Pod（`ztunnel-pods-ips` ipset 中）发起请求时，其连接会被打上 `0x100/0x100` 的标记。

一般在这种情况下，会与路由相关，我们看一下路由规则：

```console
root@ambient-worker2:/# ip rule
0:	from all lookup local
100:	from all fwmark 0x200/0x200 goto 32766
101:	from all fwmark 0x100/0x100 lookup 101
102:	from all fwmark 0x40/0x40 lookup 102
103:	from all lookup 100
32766:	from all lookup main
32767:	from all lookup default
```

可以看到，被标记了 `0x100/0x100` 的流量会走 table 101 的路由表，我们可以查看路由表：

```console
root@ambient-worker2:/# ip r show table 101
default via 192.168.127.2 dev istioout
10.244.1.2 dev veth5db63c11 scope link
```

可以明显看到，默认网关被换成了 `192.168.127.2`，且走了 istioout 网卡。

这里就有问题了，`192.168.127.2` 这个 IP 并不属于 NodeIP、PodIP、ClusterIP 中的任意一种，istioout 网卡默认应该也不存在，那这个 IP 是谁创建的呢？
因为流量最终需要发往 ztunnel，我们可以查看 ztunnel 的配置，看看能否找到答案。

```console
kebe@pc $ kubectl -n istio-system get po ztunnel-vxv4b -o yaml
apiVersion: v1
kind: Pod
metadata:
  ...
  name: ztunnel-vxv4b
  namespace: istio-system
	...
spec:
  ...
  initContainers:
  - command:
			...
      OUTBOUND_TUN=istioout
			...
      OUTBOUND_TUN_IP=192.168.127.1
      ZTUNNEL_OUTBOUND_TUN_IP=192.168.127.2

      ip link add name p$INBOUND_TUN type geneve id 1000 remote $HOST_IP
      ip addr add $ZTUNNEL_INBOUND_TUN_IP/$TUN_PREFIX dev p$INBOUND_TUN

      ip link add name p$OUTBOUND_TUN type geneve id 1001 remote $HOST_IP
      ip addr add $ZTUNNEL_OUTBOUND_TUN_IP/$TUN_PREFIX dev p$OUTBOUND_TUN

      ip link set p$INBOUND_TUN up
      ip link set p$OUTBOUND_TUN up
      ...
```

如上，ztunnel 会负责创建 istioout 网卡，我们现在去节点上查看对应网卡。

```console
root@ambient-worker2:/# ip a
11: istioout: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    link/ether 0a:ea:4e:e0:8d:26 brd ff:ff:ff:ff:ff:ff
    inet 192.168.127.1/30 brd 192.168.127.3 scope global istioout
       valid_lft forever preferred_lft forever
```

那 `192.168.127.2` 这个网关 IP 在哪呢？它被分配在了 ztunnel 里面。

```console
kebe@pc $ kubectl -n istio-system exec -it ztunnel-nptf6 -- ip a
Defaulted container "istio-proxy" out of: istio-proxy, istio-init (init)
2: eth0@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 46:8a:46:72:1d:3b brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.244.2.3/24 brd 10.244.2.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::448a:46ff:fe72:1d3b/64 scope link
       valid_lft forever preferred_lft forever
4: pistioout: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether c2:d0:18:20:3b:97 brd ff:ff:ff:ff:ff:ff
    inet 192.168.127.2/30 scope global pistioout
       valid_lft forever preferred_lft forever
    inet6 fe80::c0d0:18ff:fe20:3b97/64 scope link
       valid_lft forever preferred_lft forever
```

现在可以看到，流量会到 ztunnel 里面，但此时并没有对流量做任何其它操作，只是简单地路由到了 ztunnel。如何才能让 ztunnel 里面的 Envoy 对流量进行处理呢？

我们继续看一看 ztunnel 的配置，其中写了很多 iptables 规则。我们可以进入 ztunnel 看一下具体的规则：

```console
kebe@pc $ kubectl -n istio-system exec -it ztunnel-nptf6 -- iptables-save
Defaulted container "istio-proxy" out of: istio-proxy, istio-init (init)
...
*mangle
-A PREROUTING -i pistioout -p tcp -j TPROXY --on-port 15001 --on-ip 127.0.0.1 --tproxy-mark 0x400/0xfff
...
COMMIT
```

现在可以看到，当流量进入 ztunnel 时，会使用 TPROXY 将流量转入 15001 端口进行处理，此处的 15001 即为 Envoy 实际监听用于处理 Pod 出口流量的端口。
关于 TPROXY，大家可以自行学习相关信息，本文不再赘述。

所以，总结下来，当 Pod 处于 Ambient 模式下，其出口流量路径大致为：

1. 从 Pod 里面的进程发起流量。
2. 流量流经所在节点网络，经节点的 iptables 进行标记。
3. 节点上的路由表会将流量转发到当前节点的 ztunnel Pod。
4. 流量到达 ztunnel 时，会经过 iptables 进行 TPROXY 透明代理，将流量交给当前 Pod 中的 Envoy 的 15001 端口进行处理。

到此我们可以看出，在 Ambient 模式下，对于 Pod 出口流量的处理相对复杂。路径也比较长，不像 Sidecar 模式，直接在 Pod 内部完成流量转发。

## 入口流量拦截

有了上面的经验，我们不难发现，Ambient 模式下，对于流量的拦截主要通过 MARK 路由 + TPROXY 的方式，入口流量应该也差不多。

我们采用最简单的方式分析一下。当节点上的进程，或者其他主机上的程序相应访问当前节点上的 Pod 时，流量会经过主机的路由表。
我们查看一下当响应访问 productpage-v1-7c548b785b-w9zl6(10.244.1.7) 时的路由信息：

```console
root@ambient-worker2:/# ip r get 10.244.1.7
10.244.1.7 via 192.168.126.2 dev istioin table 100 src 10.244.1.1 uid 0
    cache
```

我们可以看到，当访问 `10.244.1.7` 时，流量会被路由到 `192.168.126.2`，而这条规则正是由上面 istio-cni 添加的。

同样地 `192.168.126.2` 这个 IP 属于 ztunnel：

```console
kebe@pc $ kubectl -n istio-system exec -it ztunnel-nptf6 -- ip a
Defaulted container "istio-proxy" out of: istio-proxy, istio-init (init)
2: eth0@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 46:8a:46:72:1d:3b brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.244.2.3/24 brd 10.244.2.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::448a:46ff:fe72:1d3b/64 scope link
       valid_lft forever preferred_lft forever
3: pistioin: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 7e:b2:e6:f9:a4:92 brd ff:ff:ff:ff:ff:ff
    inet 192.168.126.2/30 scope global pistioin
       valid_lft forever preferred_lft forever
    inet6 fe80::7cb2:e6ff:fef9:a492/64 scope link
       valid_lft forever preferred_lft forever
```

按照相同的分析方法，我们看一下 iptables 规则：

```console
kebe@pc $ kubectl -n istio-system exec -it ztunnel-nptf6 -- iptables-save
...
-A PREROUTING -i pistioin -p tcp -m tcp --dport 15008 -j TPROXY --on-port 15008 --on-ip 127.0.0.1 --tproxy-mark 0x400/0xfff
-A PREROUTING -i pistioin -p tcp -j TPROXY --on-port 15006 --on-ip 127.0.0.1 --tproxy-mark 0x400/0xfff
...
```

如果直接在节点上访问 PodIP + Pod 端口，流量会被转发到 ztunnel 的 15006 端口，而这就是 Istio 处理入口流量的端口。

至于目标端口为 15008 端口的流量，这是 ztunnel 用来做四层流量隧道的端口。本文暂不细述。

## 对于 Envoy 自身的流量处理

我们知道，在 Sidecar 模式下，Envoy 和业务容器运行在相同的网络命名空间中。对于业务容器的流量，我们需要全部拦截，以保证对流量的完全掌控，但是在 Ambient 模式下是否需要呢？

答案是否定的，因为 Envoy 已经被独立到其它 Pod 中，Envoy 发出的流量是不需要特殊处理的。换言之，对于 ztunnel，我们只需要处理入口流量即可，所以 ztunnel 中的规则看起来相对简单。

## 未完待续…

上面我们主要分析了在 Ambient 模式下对于 Pod 流量拦截的方案，还没有涉及到七层流量的处理以及 ztunnel 实现的具体原理，后续将分析流量在 ztunnel 和 waypoint proxy 中详细的处理路径。
