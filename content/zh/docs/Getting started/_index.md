---
title: "快速开始"
linkTitle: "快速开始"
weight: 2
description: >
  本文将帮助您快速开始使用 Merbridge
---

## 先决条件 {#prerequisites}

1. 系统的内核版本应大于等于 `5.7`，可以使用 `uname -r` 查看。
1. 系统应开启 cgroup2，可以通过 `mount | grep cgroup2` 进行验证。

## 安装 {#installation}

目前支持在 Istio 和 Linkerd2 环境下安装 Merbridge。

### Istio 环境 {#installation-on-istio}

只需要在环境中执行以下命令即可安装 Merbridge：

```bash
kubectl apply -f https://raw.githubusercontent.com/merbridge/merbridge/main/deploy/all-in-one.yaml
```

### Linkerd2 环境 {#installation-on-linkerd}

只需要在环境中执行以下命令即可安装 Merbridge：

```bash
kubectl apply -f https://raw.githubusercontent.com/merbridge/merbridge/main/deploy/all-in-one-linkerd.yaml
```

### Kuma 环境 {#installation-on-kuma}

只需要在环境中执行以下命令即可安装 Merbridge：

```bash
kubectl apply -f https://raw.githubusercontent.com/merbridge/merbridge/main/deploy/all-in-one-kuma.yaml
```

## 验证 {#verification}

### 验证安装 {#verification-installation}

在验证 Merbridge 是否能正常工作之前，需要先确保 Merbridge 的 Pod 都运行正常。以 Istio 为例，可以使用以下命令查看 Merbridge 的 Pod 状态：

```bash
kubectl -n istio-system get pods
```

当 Merbridge 相关的所有 Pod 都处于 Running 状态时，表明 Merbridge 已经安装成功。

### 连接测试 {#verification-connection-test}

可以按照如下方案验证 Merbridge 的连接是否正常：

#### 安装 sleep 和 helloworld 应用并等待其完全启动

```bash
kubectl label ns default istio-injection=enabled
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/sleep/sleep.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/helloworld/helloworld.yaml
```

#### 执行 curl 测试

```bash
kubectl exec $(kubectl get po -l app=sleep -o=jsonpath='{..metadata.name}') -c sleep -- curl -s -v helloworld:5000/hello
```

如果在结果中看到类似 `* Connected to helloworld (127.128.0.1) port 5000 (#0)` 的字样（其中ip是127开头），表明 Merbridge 已经成功使用 eBPF 代替 iptables 进行流量转发。
