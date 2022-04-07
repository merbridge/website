---
title: "Overview"
linkTitle: "Overview"
weight: 1
description: >
  This page provides an overview of Merbridge.
---

{{% pageinfo %}}
Obtain a general view of Merbridge.
{{% /pageinfo %}}

## What is Merbridge

Merbridge is designed to make traffic interception and forwarding more efficient for service meshes by replacing iptables with eBPF.

## Why Merbridge

In the service mesh scenario, in order to use sidecars for silent traffic management, we need to forward the inbound and outbound traffic of a Pod to sidecars.

The most common solution to this is to use the REDIRECT feature of iptables (netfilter) to redirect the original traffic to a sidecar for traffic management and other purposes.

For example, the inbound traffic that originally flows directly to the application is now forwarded by iptables (netfilter) to the sidecar, which then forwards the traffic to its destination.

However, netfilter is not performing this job well. eBPF's ability for similar purposes makes Merbridge possible.

We hope to replace iptables (netfilter) with eBPF to make service meshes more efficient.

## When to use Merbridge

Merbridge is recommended if you have following problems:

1. iptables increase latency when high-performance connection is needed.
2. Your system cannot use iptables for certain reasons.
3. Ordinary Pods cannot have too many permissions for certain reasons (e.g., the init container may require NET_ADMIN and other permissions).