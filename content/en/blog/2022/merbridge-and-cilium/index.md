---
title: "Merbridge and Cilium"
linkTitle: "Merbridge and Cilium"
date: 2022-04-23
weight: 1
---

# Merbridge and Cilium

[Cilium](https://cilium.io/) is a great open source software that provides a lot of networking capabilities for cloud native applications based on eBPF, with a lot of great designs. Among others, Cilium designed a set of sockmap-based redir capabilities to help accelerate network communications, which inspired us and is the basis for Merbridge to provide network acceleration. It is a really great design.

Merbridge leverages the great foundation that Cilium has provided, along with some targeted adaptations we've made in the Service Mesh, to make it easier to apply eBPF technology to Service Mesh.

Our development team have learned a lot eBPF theoretical knowledge, practical methods, and testing methods, from Cilium's detailed documentation and our frequent exchanges with the Cilium technical team. All these together helps make Merbridge possible.

Thanks again to the Cilium project and community, and to Cilium for these great designs.
