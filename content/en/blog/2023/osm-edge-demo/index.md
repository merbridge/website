---
title: "Test OSM Edge Integration with Merbridge"
linkTitle: "Test OSM Edge Integration with Merbridge"
date: 2023-04-03
weight: 1
---

This page will walk you through how to integrate Merbridge into [OSM Edge](https://github.com/flomesh-io/osm-edge) for mesh acceleration.

> This demo originated from [cybwan's personal space](https://github.com/cybwan/osm-edge-start-demo/blob/main/demo/merbridge/README.zh.md).

## 1. Deploy k8s

### 1.1 Preparation

- [x] Deploy 3 virtual machines of **ubuntu 22.04/20.04**, one as master node and the other two as worker nodes.
- [x] Name them as  `master`, `node1`, and `node2`.
- [x] Modify `/etc/hosts` to enable hostname-based connectivity between the three nodes.
- [x] Update apt packages：

  ```bash
  sudo apt -y update && sudo apt -y upgrade
  ```

- [x] Use root account to run the following commands.

### 1.2  Deploy resources for container initialization on each VM

```bash
curl -L https://raw.githubusercontent.com/cybwan/osm-edge-scripts/main/scripts/install-k8s-node-init.sh -O
chmod u+x install-k8s-node-init.sh

system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
./install-k8s-node-init.sh ${arch} ${system}
```

### 1.3 Deploy k8s tools on each VM

```bash
curl -L https://raw.githubusercontent.com/cybwan/osm-edge-scripts/main/scripts/install-k8s-node-init-tools.sh -O
chmod u+x install-k8s-node-init-tools.sh

system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
./install-k8s-node-init-tools.sh ${arch} ${system}

source ~/.bashrc
```

### 1.4 Launch k8s-related services on the master node

```bash
curl -L https://raw.githubusercontent.com/cybwan/osm-edge-scripts/main/scripts/install-k8s-node-master-start.sh -O
chmod u+x install-k8s-node-master-start.sh

# replace with the IP of your master node
MASTER_IP=192.168.127.80
# Set flannel as CNI
CNI=flannel
./install-k8s-node-master-start.sh ${MASTER_IP} ${CNI}
# Wait a while...
```

### 1.5 Launch k8s-related services on the worker nodes

```bash
curl -L https://raw.githubusercontent.com/cybwan/osm-edge-scripts/main/scripts/install-k8s-node-worker-join.sh -O
chmod u+x install-k8s-node-worker-join.sh

# replace with the IP of your master node
MASTER_IP=192.168.127.80
# enter root passwords of Master node as required
./install-k8s-node-worker-join.sh ${MASTER_IP}
```

### 1.6 Check status of k8s-related pods on the master node

```bash
kubectl get pods -A -o wide
```

## 2. Deploy osm-edge

### 2.1 Download and install osm-edge CLT

```bash
system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
release=v1.3.3
curl -L https://github.com/flomesh-io/osm-edge/releases/download/${release}/osm-edge-${release}-${system}-${arch}.tar.gz | tar -vxzf -
./${system}-${arch}/osm version
cp ./${system}-${arch}/osm /usr/local/bin/
```

### 2.2 Install osm-edge

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

If you want to deploy `osm`, use the following commands:

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

## 3. Deploy Merbridge

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

## 4. Test if Merbridge replaces iptables

### 4.1 Deploy business pods

```bash
# simulate business services
kubectl create namespace demo
osm namespace add demo
kubectl apply -n demo -f https://raw.githubusercontent.com/istio/istio/master/samples/sleep/sleep.yaml
kubectl apply -n demo -f https://raw.githubusercontent.com/istio/istio/master/samples/helloworld/helloworld.yaml

# schedule pods to different nodes
kubectl patch deployments sleep -n demo -p '{"spec":{"template":{"spec":{"nodeName":"node1"}}}}'
kubectl patch deployments helloworld-v1 -n demo -p '{"spec":{"template":{"spec":{"nodeName":"node1"}}}}'
kubectl patch deployments helloworld-v2 -n demo -p '{"spec":{"template":{"spec":{"nodeName":"node2"}}}}'

# wait supportive pods to run
kubectl wait --for=condition=ready pod -n demo -l app=sleep --timeout=180s
kubectl wait --for=condition=ready pod -n demo -l app=helloworld -l version=v1 --timeout=180s
kubectl wait --for=condition=ready pod -n demo -l app=helloworld -l version=v2 --timeout=180s
```

### 4.2 Scenarios Test

#### 4.2.1 Monitor kernel logs on node1 and node2

```bash
cat /sys/kernel/debug/tracing/trace_pipe|grep bpf_trace_printk|grep -E "rewritten|redirect"
```

#### 4.2.2 Command for testing

Run it multiple times:

```bash
kubectl exec $(kubectl get po -l app=sleep -n demo -o=jsonpath='{..metadata.name}') -n demo -c sleep -- curl -s helloworld:5000/hello
```

#### 4.2.3 Test results

The expected output should be like:

```bash
Hello version: v1, instance: helloworld-v1-5d46f78b4c-hghcj
Hello version: v2, instance: helloworld-v2-6b56769f9d-stwrj
```
