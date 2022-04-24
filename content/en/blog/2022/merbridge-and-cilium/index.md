---
title: "Merbridge and Cilium Hand in Hand"
linkTitle: "Merbridge and Cilium Hand in Hand"
date: 2022-04-23
weight: 1
---

# Merbridge and Cilium Hand in Hand

[Cilium](https://cilium.io/) is an excellent open source software that provides many networking capabilities for cloud native applications based on eBPF. As a pioneer in the developing roadmap of eBPF, it provides great design concepts, which also inspires our team to initiate the Merbridge project.

Merbridge is also stemmed from eBPF to provide faster and more efficient experience on connectivity for service mesh applications. The applicable scenarios are different from Cilium, but both are symbiotic. Cilium provides much possibility to enable Merbridge growing up.

Cilium mainly provides basic network capabilities for cloud native applications, such as ClusterIP, NodePort, and network policy. Merbridge is using eBPF instead of iptables to realize traffic interception and accelerate your network in the service mesh scenario. It also needs to rely on the capabilities of underlying CNI, such as Cilium.

Therefore, Merbridge only focuses on enhancing the network capability in the service mesh and needs to rely on the underlying CNI like Cilium, but it can provide users with another better choice for filtering and acceleration.

## Acknowledgements

Cilium has developed in the field of eBPF for many years and has been very successful in business operation. Cilium has promoted the eBPF technology to the open source community in various ways and output many useful blogs and workshops. These assets provide a lot of help and inspiration for the emergence of Merbridge.

Our team has learned much about the basic knowledge, practice cases and testing procedures of eBPF from the documents provided by Cilium. We also talked several times with Cilium technicians in a good manner. All these activities and experience enable the successful incubation of Merbridge.

Once again, we sincerely thanks the Cilium community for their great help and welcome you to join the [Merbridge community](https://github.com/merbridge) to discuss technical issues and promote the development and growth of eBPF technologies. I believe that Cilium and Merbridge will benefit a lot with the further evolution of eBPF.
