---
title: "Quick Start"
linkTitle: "Quick Start"
weight: 2
description: >
  This page helps you quickly get started with Merbridge.
---

## Prerequisites {#prerequisites}

1. Use kernel `5.7` or a higher version. Check your version with `name -r`.
1. Activate `cgroup2` in your system. Check the status with `mount | grep cgroup2`.

## Installation {#installation}

Merbridge can be installed on Istio and Linkerd2 only.

### Install on Istio {#installation-on-istio}

Apply the following command to install Merbridge:

```bash
kubectl apply -f https://raw.githubusercontent.com/merbridge/merbridge/main/deploy/all-in-one.yaml
```

### Install on Linkerd2 {#installation-on-linkerd}

Apply the following command to install Merbridge:

```bash
kubectl apply -f https://raw.githubusercontent.com/merbridge/merbridge/main/deploy/all-in-one-linkerd.yaml
```

## Verification {#verification}

### Verify installation {#verification-installation}

Before you start this verification, make sure each Pod of Merbridge is running well. You can check the Pod status in Istio with the following command:

```bash
kubectl -n istio-system get pods
```

If the status of all Pods related to Merbridge is Running, it means Merbridge is successfully installed.

### Verify connection {#verification-connection-test}

Use the following methods to check the connectivity of Merbridge:

#### Install sleep and helloworld and wait for a complete start:

```bash
kubectl label ns default istio-injection=enabled
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/sleep/sleep.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/helloworld/helloworld.yaml
```

#### Conduct curl test:

```bash
kubectl exec $(kubectl get po -l app=sleep -o=jsonpath='{..metadata.name}') -c sleep -- curl -s -v helloworld:5000/hello
```

If you see words like `* Connected to helloworld (127.128.0.1) port 5000 (#0)` in the output, it means Merbridge has managed to replace iptables with eBPF for traffic forwarding.

