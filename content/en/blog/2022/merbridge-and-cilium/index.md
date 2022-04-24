---
title: "Merbridge and Cilium"
linkTitle: "Merbridge and Cilium"
date: 2022-04-23
weight: 1
---

# Merbridge and Cilium

[Cilium](https://cilium.io/) is an excellent open source software that provides many networking capabilities for cloud native applications based on eBPF. As a pioneer in the developing roadmap of eBPF, it provides great design concepts, which also inspires our team to initiate the Merbridge project.

Merbridge is also stemmed from eBPF to provide faster and more efficient experience on connectivity for service mesh applications. Although the application scenarios of Merbridge are different from Cilium, Cilium provides many possibilities to unblock Merbridge's growth.

Cilium mainly provides basic network capabilities for cloud native applications, such as ClusterIP, NodePort, and network policy. Merbridge uses eBPF instead of iptables to perform traffic interception and accelerate your network in the service mesh scenario, which also relies on the capabilities of underlying CNI, such as Cilium.

Since the two projects have different motives, better eBPF solution is provided by Merbridge for users in the mesh scenario.

## Acknowledgements

Cilium has been contributed to the field of eBPF for many years and becomes very successful in business operation. It has promoted the eBPF technology to the open source community in various ways and output many useful blogs and workshops. These assets provide a lot of help and inspiration for the beginning of Merbridge.

Our team has learned a lot from the documents provided by Cilium, like practice cases, testing procedures. We also talked several times with Cilium technicians in a good manner. All these activities and experience enable the successful incubation of Merbridge.

Once again, we sincerely thanks the Cilium community for their great help and welcome you to join the [Merbridge community](https://github.com/merbridge) to discuss technical issues and promote the development and growth of eBPF technologies. We believe that Cilium and Merbridge will both benefit a lot with the further evolution of eBPF.
