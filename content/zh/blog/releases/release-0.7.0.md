---
title: "发布 0.7.0"
linkTitle: "发布 0.7.0"
date: 2022-07-20
description: >
  0.7.0 版本更新内容。
---

**我们很高兴地发布 Merbridge 0.7.0！**

本次更新主要体现在两个方面：
1. 使用 `tc`（Traffic Control）代替 XDP 来做容器的出入口流量处理，规避 XDP Generic 模式存在的问题，并且可用于生产环境。
2. 增加对 Kuma 的支持，可以在 Kuma 中使用和 Istio 相同的能力。感谢 Kuma 工程师 [@bartsmykla](https://github.com/bartsmykla) 的贡献。

有关其他更新内容，请参考：[Merbridge 0.7.0](https://github.com/merbridge/merbridge/releases/tag/0.7.0)
