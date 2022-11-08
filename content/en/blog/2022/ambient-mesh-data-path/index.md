---
title: Deep Dive into Ambient Mesh - Traffic Path
linkTitle: Deep Dive into Ambient Mesh - Traffic Path
date: 2022-10-13
weight: 1
description: This blog analyzes the traffic path on data plane in the ambient mesh.
author: Kebe Liu
---

Ambient Mesh has been released for a while, and some online articles have talked much about its usage and architecture.
This blog will dive into the traffic path on data plane in the ambient mesh to help you fully understand
the implementations of the ambient data plane.

Before start, you shall carefully read through [introducing ambient mesh](https://istio.io/latest/blog/2022/introducing-ambient-mesh/)
to learn the basic knowledge of the ambient mesh.

> For your convenience, the test environment can be deployed by following
> [Get Started with Istio Ambient Mesh](https://istio.io/latest/blog/2022/get-started-ambient/).

## Start from the moment you make a request

In order to explore the traffic path, we first analyze the scenario where two services access each other in
the ambient mesh (only for L4 mode on different nodes).

After enabling the ambient mesh in the `default` namespace, all services will have capabilities of mesh governance.

Our analysis starts from this command: `kubectl exec deploy/sleep -- curl -s http://productpage:9080/ | head -n1`

In the sidecar mode, Istio intercepts traffic through iptables.
When you run the curl command in a sleep pod, the traffic will be forwarded by iptables to port 15001 of sidecar.
However, in the ambient mesh, no sidecar exists in a pod, and it does not need restart to enable the ambient mesh.
How to make sure the request is processed by ztunnel?

## Egress traffic interception

To learn details about intercepting the egress traffic, let's check the control plane components:

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

In the sidecar mode, istio-cni is mainly a CNI to avoid permission
leakage caused by using the istio-init container to process iptables rules.
However, in the ambient mesh, istio-cni becomes a required component.
Sidecars are theoretically not needed. Why is the istio-cni component being required?

Let's check the logs:

```console
kebe@pc $ kubectl -n istio-system logs istio-cni-node-qsvsz
...
2022-10-12T07:34:33.224957Z	info	ambient	Adding route for reviews-v1-6494d87c7b-zrpks/default: [table 100 10.244.1.4/32 via 192.168.126.2 dev istioin src 10.244.1.1]
2022-10-12T07:34:33.226054Z	info	ambient	Adding pod 'reviews-v2-79857b95b-m4q2g/default' (0ff78312-3a13-4a02-b39d-644bfb91e861) to ipset
2022-10-12T07:34:33.228305Z	info	ambient	Adding route for reviews-v2-79857b95b-m4q2g/default: [table 100 10.244.1.5/32 via 192.168.126.2 dev istioin src 10.244.1.1]
2022-10-12T07:34:33.229967Z	info	ambient	Adding pod 'reviews-v3-75f494fccb-92nq5/default' (e41edf7c-a347-45cb-a144-97492faa77bf) to ipset
2022-10-12T07:34:33.232236Z	info	ambient	Adding route for reviews-v3-75f494fccb-92nq5/default: [table 100 10.244.1.6/32 via 192.168.126.2 dev istioin src 10.244.1.1]
```

As shown in the above output, for a pod in the ambient mesh, istio-cni performs the following actions:

1. Add the pod to ipset
2. Add a routing rule to table 100 (for its usage see below)

You can view the ipset contents on the node (note that the kind cluster is used here,
you need to use `docker exec` to enter the host first):

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

It is found that an ipset exists on the node where this pod is running. ipset holds many IPs for pods.

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

Therefore, this ipset holds a list of all PodIPs in the ambient mesh on the current node.

Where can this ipset be used?

Let's take a look at the iptables rules and you can find:

```console
root@ambient-worker2:/# iptables-save
*mangle
...
-A POSTROUTING -j ztunnel-POSTROUTING
...
-A ztunnel-PREROUTING -p tcp -m set --match-set ztunnel-pods-ips src -j MARK --set-xmark 0x100/0x100
```

You now learn that when a pod in the ambient mesh on a node (in the `ztunnel-pods-ips` ipset) initiates a request,
its connection will be marked with `0x100/0x100`.

Generally, it will be related to routing. Let's check the routing rules:

```console
root@ambient-worker2:/# ip rule
0: from all lookup local
100: from all fwmark 0x200/0x200 goto 32766
101: from all fwmark 0x100/0x100 lookup 101
102: from all fwmark 0x40/0x40 lookup 102
103: from all lookup 100
32766: from all lookup main
32767: from all lookup default
```

The traffic marked with `0x100/0x100` goes via the routing table 101. Let's check the routing table:

```console
root@ambient-worker2:/# ip r show table 101
default via 192.168.127.2 dev istioout
10.244.1.2 dev veth5db63c11 scope link
```

It can be clearly seen that the default gateway has been replaced with `192.168.127.2` via the istioout NIC (network interface card).

`192.168.127.2` does not belong to any of NodeIP, PodIP, and ClusterIP.
The istioout NIC should not exist by default, then where does this IP come from?
Since the traffic ultimately needs to go to ztunnel,
you can check the ztunnel configuration to see if you can find the answer.

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

As above, ztunnel will be responsible for creating the istioout NIC, you now go to the node to check the NIC.

```console
root@ambient-worker2:/# ip a
11: istioout: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    link/ether 0a:ea:4e:e0:8d:26 brd ff:ff:ff:ff:ff:ff
    inet 192.168.127.1/30 brd 192.168.127.3 scope global istioout
       valid_lft forever preferred_lft forever
```

Where is the gateway IP of `192.168.127.2`? It is allocated in ztunnel.

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

You can now see that the traffic is going to ztunnel, but nothing else is done to the traffic at this time,
it is simply routed to ztunnel. How to does Envoy in ztunnel process the traffic?

Let's continue to check the ztunnel configuration with many iptables rules. Let's check the specific rules in ztunnel.

```console
kebe@pc $ kubectl -n istio-system exec -it ztunnel-nptf6 -- iptables-save
Defaulted container "istio-proxy" out of: istio-proxy, istio-init (init)
...
*mangle
-A PREROUTING -i pistioout -p tcp -j TPROXY --on-port 15001 --on-ip 127.0.0.1 --tproxy-mark 0x400/0xfff
...
COMMIT
```

When traffic enters ztunnel, it will use TPROXY to transfer the traffic to port 15001 for processing,
where 15001 is the port that Envoy actually listens to and process the pod egress traffic.
As for TPROXY, you can learn relevant reference, and this blog will not repeat it further.

So when a pod is running in the ambient mesh, its egress traffic path is as follows:

1. Initiate traffic from a process in pod
2. The traffic flows via the node network and get marks by iptables on the node
3. The traffic is forwarded to the ztunnel pod on current node by the routing table
4. When the traffic reaches ztunnel, it will go through iptables for TPROXY (transparent proxy),
   and send the traffic to port 15001 of Envoy in the current pod.

So far in the ambient mesh, it is clear that the processing of pod egress traffic is relatively complex.
The path is also relatively long, unlike the sidecar mode in which a traffic forwarding is directly completed in the pod.

## Ingress traffic interception

With the above experience, it is easy to learn that in the ambient mesh, the traffic interception is mainly
through the method of MARK routing + TPROXY, and the ingress traffic should be similar.

Let's analyze it in the simplest way. When a process on a node, or a program on another host accesses
a pod on the current node, the traffic goes through the host's routing table.
Let's check the routing info when the response arrives at `productpage-v1-7c548b785b-w9zl6(10.244.1.7)`:

```console
root@ambient-worker2:/# ip r get 10.244.1.7
10.244.1.7 via 192.168.126.2 dev istioin table 100 src 10.244.1.1 uid 0
    cache
```

When accessing `10.244.1.7`, the traffic will be routed to `192.168.126.2`, and this rule is added by istio-cni.

Similarly `192.168.126.2` belongs to ztunnel:

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

By using the same analysis method, let's check the iptables rules:

```console
kebe@pc $ kubectl -n istio-system exec -it ztunnel-nptf6 -- iptables-save
...
-A PREROUTING -i pistioin -p tcp -m tcp --dport 15008 -j TPROXY --on-port 15008 --on-ip 127.0.0.1 --tproxy-mark 0x400/0xfff
-A PREROUTING -i pistioin -p tcp -j TPROXY --on-port 15006 --on-ip 127.0.0.1 --tproxy-mark 0x400/0xfff
...
```

If you directly access the node via PodIP + pod port, the traffic will be forwarded to port 15006 of ztunnel,
which is the port to handle the ingress traffic in Istio.

As for the traffic whose destination port is port 15008, this is the port used by ztunnel for L4 traffic tunneling.
This blog will not explain this further.

## Handle traffic for Envoy itself

In the sidecar mode, Envoy and user containers run in the same network namespace.
For the traffic from user containers, you need to intercept all the traffic to
guarantee complete control of the traffic. However, is it also required in the ambient mesh?

The answer is no, because Envoy has been isolated from other pods.
The traffic sent by Envoy does not require special notice.
In other words, you only need to handle ingress traffic for ztunnel,
so the rules in ztunnel seem relatively simple.

## Wrapping up

As above explained, this blog mainly analyzed the scheme for Pod traffic interception in the ambient mesh,
but this blog has not involved with how to handle L7 traffic and the specific principles of ztunnel implementation.
The next plan is to analyze the detailed traffic paths in ztunnel and waypoint proxy.
