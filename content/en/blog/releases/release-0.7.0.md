---
title: "Release 0.7.0"
linkTitle: "Release 0.7.0"
date: 2022-07-20
description: >
  What is new in version 0.7.0?
---

**We are pleased to announce the release of Merbridge 0.7.0!**

This release mainly includes two updates as below:

1. Replaced XDP with `tc` (Traffic Control) to handle the ingress/egress traffic. This replacement comes from a situation that the XDP Generic mode has some problems and is not recommended to use in a production environment.

2. Merbridge now supports [Kuma](https://kuma.io/), a universal Envoy service mesh. Similar to Istio, you can use all capabilities provided by Merbridge when you are working in the Kuma environment. Thanks [@bartsmykla](https://github.com/bartsmykla), an excellent engineer from Kuma.

For release notes, see [Merbridge 0.7.0](https://github.com/merbridge/merbridge/releases/tag/0.7.0).
