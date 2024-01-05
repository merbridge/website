---
title: "Concepts"
linkTitle: "Concepts"
weight: 4
description: >
  This page describes some key concepts about Merbridge.
---

## eBPF

The full name of eBPF is Extended Berkeley Packet Filter. As the name implies,
this is a module used to filter network packets. For example, sockops and redir
capabilities of eBPF can efficiently filter and intercept packets.

eBPF is a revolutionary technology with origins in the Linux kernel that can run
sandboxed programs in an operating system kernel. It is used to safely and efficiently
extend the capabilities of the kernel without requiring to change kernel source code or load kernel modules.

## iptables

iptables is a traffic filter built on netfilter. It implements traffic filtering and interception
by registering hook functions on the mount point of netfilter. From the name of iptables, we can
guess it may contain some tables. In practice, by mounting rule tables on different chains of
netfilter, iptables can filter or modify the traffic packets entering and leaving the kernel protocol stack.

iptables has 4 tables by default:

- Filter
- NAT
- Raw
- Mangle

iptables has 5 chains by default:

- INPUT chain (ingress rules)
- OUTPUT chain (egress rules)
- FORWARD chain (rules of forwarding)
- PREROUTING chain (rules before routing)
- POSTROUTING chain (rules after routing)

## Service Mesh

A service mesh is a dedicated infrastructure layer for handling service-to-service communication.
It’s responsible for the reliable delivery of requests through the complex topology of services that
comprise a modern, cloud native application. A service mesh can guarantee fast, reliable, and secure
communication between containerized application infrastructure services. Key capabilities provided by
mesh include service discovery, load balancing, secure encryption and authentication, failover,
observability, and more.

A service mesh typically injects a sidecar proxy into each service instance. These sidecars handle
inter-service communication, monitoring, and security. In this way, developers can focus on the
development, support, and maintenance of the application code in the service, while the O\&M team
is responsible for the maintenance of the service mesh and applications.

Today, the most well-known service mesh is Istio.

## Istio

Istio is a service mesh technology originally open sourced by IBM, Google, and Lyft. It can be layered
transparently onto distributed applications and provides all the benefits of a service mesh, such as
traffic governance, security, and observability.

Istio can adapt to all services hosted with on-premises, cloud, Kubernetes containers, and virtual machines.
It is typically used with microservices deployed on a Kubernetes platform.

Fundamentally, Istio works by deploying an extended version of Envoy as a sidecar proxy to each microservice.
The proxy network it uses forms a data plane of Istio. The configuration and management of these proxies is
done in a control plane, providing discovery, configuration, and certificate management for Envoy proxies
in the data plane.

## Linkerd

Linkerd is the first service mesh launched on the market, but Istio is more popular today.

Linkerd is an open source, ultra-lightweight service mesh designed by Buoyant for Kubernetes.
It is completely rewritten in Rust, which makes it as small, light and safe as possible.
It provides runtime debugging, observability, reliability, and safety without code changes
in distributed applications.

Linkerd has three basic components: UI, data plane, and control plane. Linkerd works by installing
a set of ultra-light, transparent proxies next to each service instance that automatically handle
all traffic to and from the service.
