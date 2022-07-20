---
title: "发布 0.7.0"
linkTitle: "发布 0.7.0"
date: 2022-07-20
description: >
  0.7.0 版本更新内容。
---

**我们很高兴地发布 Merbridge 0.7.0！**

本次更新主要体现在两个方面：
一方面是使用 `tc`（Traffic Control）代替 XDP 来做容器的出入口的流量处理。其主要原因是之前所使用的 XDP Generic 模式存在一些问题，且不推荐生产使用。
另一方面，在 Kuma 工程师 [@bartsmykla](https://github.com/bartsmykla) 的帮助下，Merbridge 增加了对 Kuma 的支持，可以在 Kuma 中使用和 Istio 相同的能力。

其它改动，请参考：[Merbridge 0.7.0](https://github.com/merbridge/merbridge/releases/tag/0.7.0)
