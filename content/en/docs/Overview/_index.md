---
title: "Overview"
linkTitle: "Overview"
weight: 1
description: >
  This page provides an overview of Merbridge.
---

## What is Merbridge

Merbridge is designed to make traffic interception and forwarding more efficient for service meshes by replacing iptables with eBPF.

eBPF (extended Berkeley Packet Filter) can run user's programs in the Linux kernel without modifying the kernel code or loading kernel modules. It is widely used in networking, security, monitoring and other fields. Compared with iptables, Merbridge can shorten the data path between sidecars and services and therefore accelerate network. Meanwhile, Merbridge will not change the original architecture of Istio, and the original logic is still smooth. This means that if you don't want Merbridge anymore, just delete the DaemonSet and return to the traditional iptables. No more actions will be needed.

## What Merbridge can do

Merbridge has following core features:

- Processing outbound traffic

Merbridge uses eBPF’s `connect` program to modify `user_ip` and `user_port`, so as to modify the destination address of a connection and ensure traffic can be sent to the new interface. In order to make Envoy identify the original destination address, the application (incl. Envoy) will call the `get_sockopt` function to obtain `ORIGINAL_DST` when receiving a connection.

- Processing inbound traffic

The processing of inbound traffic is basically similar to outbound traffic. Note that eBPF cannot take effect in a specified namespace like iptables. The changes will be global. It means that if we use a Pod that is not originally managed by Istio, or an external IP address, serious problems will be encountered — like the connection not being established at all.

As a result, we designed a tiny control plane (deployed as a DaemonSet), which watches all pods — similar to the kubelet watching pods on the node — to write the pod IP addresses that have been injected into the sidecar to the `local_pod_ips` map. If the destination address is not in the map, Merbridge will not do anything to the original traffic.

- Accelerating 

In Istio, Envoy accesses the application by using the current pod IP and port number. Because the pod IP exists in the `local_pod_ips` map as well, the traffic will be redirected to the pod IP on port 15006, producing an infinite loop. Are there any ways to get the IP address in the current namespace with eBPF? Yes! We have designed a feedback mechanism: When Envoy tries to establish a connection, we redirect it to port 15006. However, in the sockops step, we will determine if the source IP and the destination IP are the same. If yes, it means the wrong request is sent, and we will discard this connection in the sockops process. In the meantime, the current ProcessID and IP information will be written into the `process_ip map` to allow eBPF to support corresponding relationship between processes and IPs. When the next request is sent, We will check directly from the `process_ip map` if the destination address is the same as the current IP address. Envoy will retry when the request fails, and this retry process will only occur once, meaning subsequent requests will be accelerated.

## Why need Merbridge

In the service mesh scenario, in order to use the sidecar for traffic management without the application being aware of it, the ingress and egress traffic of the Pod will be forwarded to the sidecar. The most common way for this is using the redirect capability of iptables (netfilter) to forward the original traffic to the sidecar. However, this approach will increase network latency, because iptables intercepts both egress and ingress traffic. Take ingress traffic as an example. The traffic that originally flows directly to the application needs to be forwarded to the sidecar by iptables (netfilter), and then the sidecar will forward the traffic to the actual application. The data path becomes very long, since duplicated steps are performed several times.

Luckily, eBPF provides a function bpf_msg_redirect_hash, to directly forward the packets sent by the application in the inbound socket to the outbound socket. By doing so, packet processing can be greatly accelerated in the kernel. Therefore, we hope to replace iptables with eBPF. That's how Merbridge came into being.

## When to use Merbridge

Merbridge is recommended if you have following problems:

- iptables takes effect globally and related modifications cannot be explicitly prohibited, resulting in poor controllability.
- When large concurrency and high-performance connections are required, using iptables will increase latency.
- Your system cannot use iptables for certain reasons.
- You cannot give ordinary Pods too many permissions for certain reasons (e.g., the init container may require NET_ADMIN and other permissions).
- Using iptables for transparent interception requires connection tracking feature of the conntrack module. This means great overhead and alos maybe a full track table in the case of a large number of connections.

## What Merbridge will change

Using eBPF to process connections on the host can greatly simplify the kernel's processing of traffic and improve the communication quality between services.

- Before applying eBPF using Merbridge, the data path between pods is like:

![iptable path](./imgs/iptables_Path.png)

- After applying Merbridge, the outbound traffic will skip many filter steps to improve the performance:

![eBPF path](./imgs/eBPF_Path.png)

- If two pods are on the same node, the connection can even be faster:

![same-node eBPF path](./imgs/sameNode_eBPF_Path.png)

[Merbridge](https://github.com/merbridge/merbridge) is a completely independent open source project. It is still at an early stage, and we are looking forward to having more users and developers to get engaged. It would be greatly appreciated if you would try this new technology to accelerate your mesh, and provide us with some feedback!　