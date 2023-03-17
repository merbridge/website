---
title: "OSM Edge 与 Merbridge 集成测试"
linkTitle: "OSM Edge 与 Merbridge 集成测试"
date: 2023-03-18
weight: 1
---

本文演示了如何在 [OSM Edge](https://github.com/flomesh-io/osm-edge) 边缘网格环境中集成 Merbridge 实现网格加速效果。

> 这篇 Demo 集成测试源于 [cybwan 个人空间](https://github.com/cybwan/osm-edge-start-demo/blob/main/demo/merbridge/README.zh.md)。

## 1. 部署 k8s 环境

### 1.1 部署环境准备

- [x] 部署 3 个 **ubuntu 22.04/20.04** 的虚机，一个作为 master 节点，两个作为 worker 节点
- [x] 主机名分别设置为 master、node1、node2
- [x] 修改 `/etc/hosts`，使其相互间可以通过主机名互通
- [x] 更新系统软件包：

  ```bash
  sudo apt -y update && sudo apt -y upgrade
  ```

- [x] 以 root 身份执行后续部署指令

### 1.2 各虚机上部署容器环境

```bash
curl -L https://raw.githubusercontent.com/cybwan/osm-edge-scripts/main/scripts/install-k8s-node-init.sh -O
chmod u+x install-k8s-node-init.sh

system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
./install-k8s-node-init.sh ${arch} ${system}
```

### 1.3 各虚机上部署 k8s 工具

```bash
curl -L https://raw.githubusercontent.com/cybwan/osm-edge-scripts/main/scripts/install-k8s-node-init-tools.sh -O
chmod u+x install-k8s-node-init-tools.sh

system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
./install-k8s-node-init-tools.sh ${arch} ${system}

source ~/.bashrc
```

### 1.4 Master 节点启动 k8s 相关服务

```bash
curl -L https://raw.githubusercontent.com/cybwan/osm-edge-scripts/main/scripts/install-k8s-node-master-start.sh -O
chmod u+x install-k8s-node-master-start.sh

# 调整为你的 master 的 ip 地址
MASTER_IP=192.168.127.80
# 使用 flannel 网络插件
CNI=flannel
./install-k8s-node-master-start.sh ${MASTER_IP} ${CNI}
# 耐心等待..
```

### 1.5 node1 和 node2 启动 k8s 相关服务

```bash
curl -L https://raw.githubusercontent.com/cybwan/osm-edge-scripts/main/scripts/install-k8s-node-worker-join.sh -O
chmod u+x install-k8s-node-worker-join.sh

# 调整为你的 master 的 ip 地址
MASTER_IP=192.168.127.80
# 安装过程会提示输入 master 的 root 的密码
./install-k8s-node-worker-join.sh ${MASTER_IP}
```

### 1.6 Master 节点查看 k8s 相关服务的启动状态

```bash
kubectl get pods -A -o wide
```

## 2. 部署 osm-edge 服务

### 2.1 下载并安装 osm-edge 命令行工具

```bash
system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
release=v1.3.3
curl -L https://github.com/flomesh-io/osm-edge/releases/download/${release}/osm-edge-${release}-${system}-${arch}.tar.gz | tar -vxzf -
./${system}-${arch}/osm version
cp ./${system}-${arch}/osm /usr/local/bin/
```

### 2.2 安装 osm-edge

```bash
export osm_namespace=osm-system
export osm_mesh_name=osm

osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.image.registry=flomesh \
    --set=osm.image.tag=1.3.3 \
    --set=osm.certificateProvider.kind=tresor \
    --set=osm.image.pullPolicy=Always \
    --set=osm.enablePermissiveTrafficPolicy=true \
    --set=osm.controllerLogLevel=warn \
    --timeout=900s
```

如果部署 osm，指令参考:

```bash
system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
release=v1.2.3
curl -L https://github.com/openservicemesh/osm/releases/download/${release}/osm-${release}-${system}-${arch}.tar.gz | tar -vxzf -
./${system}-${arch}/osm version
cp ./${system}-${arch}/osm /usr/local/bin/

export osm_namespace=osm-system
export osm_mesh_name=osm

osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.image.registry=openservicemesh \
    --set=osm.image.tag=v1.2.3 \
    --set=osm.certificateProvider.kind=tresor \
    --set=osm.image.pullPolicy=Always \
    --set=osm.enablePermissiveTrafficPolicy=true \
    --set=osm.controllerLogLevel=warn \
    --verbose \
    --timeout=900s
```

## 3. 部署 Merbridge 服务

```bash
curl -L https://raw.githubusercontent.com/merbridge/merbridge/main/deploy/all-in-one-osm.yaml -O
sed -i 's/--cni-mode=false/--cni-mode=true/g' all-in-one-osm.yaml
sed -i '/--cni-mode=true/a\\t\t- --debug=true' all-in-one-osm.yaml
sed -i 's/\t/    /g' all-in-one-osm.yaml
kubectl apply -f all-in-one-osm.yaml

sleep 5s
kubectl wait --for=condition=ready pod -n osm-system -l app=merbridge --field-selector spec.nodeName==master --timeout=1800s
kubectl wait --for=condition=ready pod -n osm-system -l app=merbridge --field-selector spec.nodeName==node1 --timeout=1800s
kubectl wait --for=condition=ready pod -n osm-system -l app=merbridge --field-selector spec.nodeName==node2 --timeout=1800s
```

## 4. Merbridge 替代 iptables 测试

### 4.1 部署业务 Pod

```bash
# 模拟业务服务
kubectl create namespace demo
osm namespace add demo
kubectl apply -n demo -f https://raw.githubusercontent.com/istio/istio/master/samples/sleep/sleep.yaml
kubectl apply -n demo -f https://raw.githubusercontent.com/istio/istio/master/samples/helloworld/helloworld.yaml

# 让 Pod 分布到不同的 node 上
kubectl patch deployments sleep -n demo -p '{"spec":{"template":{"spec":{"nodeName":"node1"}}}}'
kubectl patch deployments helloworld-v1 -n demo -p '{"spec":{"template":{"spec":{"nodeName":"node1"}}}}'
kubectl patch deployments helloworld-v2 -n demo -p '{"spec":{"template":{"spec":{"nodeName":"node2"}}}}'

# 等待依赖的 Pod 正常启动
kubectl wait --for=condition=ready pod -n demo -l app=sleep --timeout=180s
kubectl wait --for=condition=ready pod -n demo -l app=helloworld -l version=v1 --timeout=180s
kubectl wait --for=condition=ready pod -n demo -l app=helloworld -l version=v2 --timeout=180s
```

### 4.2 场景测试一

#### 4.2.1 在 node1 和 node2 上监测内核日志

```bash
cat /sys/kernel/debug/tracing/trace_pipe|grep bpf_trace_printk|grep -E "rewritten|redirect"
```

#### 4.2.2 测试指令

多次执行：

```bash
kubectl exec $(kubectl get po -l app=sleep -n demo -o=jsonpath='{..metadata.name}') -n demo -c sleep -- curl -s helloworld:5000/hello
```

#### 4.2.3 测试结果

正确返回结果类似于：

```bash
Hello version: v1, instance: helloworld-v1-5d46f78b4c-hghcj
Hello version: v2, instance: helloworld-v2-6b56769f9d-stwrj
```
