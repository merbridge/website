---
title: Merbridge now supports Ambient Mesh, no worry about CNI compatibility!
linkTitle: Merbridge now supports Ambient Mesh, no worry about CNI compatibility!
date: 2022-11-11
weight: 1
description: This blog describes how Merbridge supports Ambient Mesh.
author: Kebe Liu
---

In the blog [Deep Dive into Ambient Mesh - Traffic Path](../ambient-mesh-data-path/index.md),
we analyzed how Ambient Mesh forwards the ingress and egress traffic of Pod to ztunnel.
It is implemented by iptables + TPROXY + routing table. The traffic datapath is relatively
long compared to sidecar mode, and the principle is complicated. Moreover, it uses routing marks, which may cause
unexpected behaviors in some cases when it relies on CNI or it is running in a CNI with the bridge mode.
These severely limit the applicable scope of ambient mesh.

The main purpose of Merbridge is to replace iptables with eBPF to accelerate applications running in
a service mesh. Ambient Mesh is a new mode of Istio. It is necessary for Merbridge to support this new mode.
iptables is a powerful tool to block unwanted traffic, allow desired traffic, and redirect packets to specific
addresses and ports, but it also has some weaknesses. First, iptables uses a linear matching method.
When several applications simultaneously call a same program, conflicts may arise and make some features become unavailable.
Second, although it is flexible enough, it still cannot be programmed as freely as eBPF.
Therefore, replacing iptables with eBPF can help Ambient Mesh achieve traffic interception.

## Objectives

As mentioned in [Deep Dive into Ambient Mesh - Traffic Path](../ambient-mesh-data-path/index.md),
we set two objectives:

- Outgoing traffic from pods in Ambient Mesh should be intercepted and redirected to port 15001 of ztunnel.
- Traffic sent from host applications to pods in Ambient Mesh should be redirected to port 15006 of ztunnel.

Since istioin and other network interface cards (NICs) are completely designed to adapt to the native Ambient Mesh, we don't need to make any changes.

## Pain points analysis

Ambient Mesh has a different operation mechanism from sidecars. According to the official definition of Istio,
adding a Pod to Ambient Mesh does not require restarting the Pod and no any sidecar-related process is running in the Pod. It means:

1. Merbridge used the CNI mode to enable the eBPF program to get the current Pod IP to make policy decisions,
   which is incompatible with ambient mesh. The reason is a pod will not be restarted after joining or
   leaving the ambient mesh, nor will it call the CNI plug-in.
2. In the sidecar mode, the only thing you need to change is the destination address to `127.0.0.1:15001` in
   the connect hook of eBPF, but in the ambient mesh you need to replace the desitination IP with that of ztunnel.

In addtion, no sidecar-related process exists in a Pod running in an ambient mesh,
so the legacy method of checking whether a port such as 15006 is listening in the current Pod is no longer applicable.
It is necessary to redesign the scheme to check the environment where processes are running.

Therefore, based on the above analysis, it is required to redesign the entire interception scheme so that Merbridge can support the ambient mesh.

In summary, we need to implement the following features:

- Redesign a scheme for judging whether a Pod is running in the ambient mesh
- Use eBPF to perceive current Pod IP regardless of CNIs
- Enable eBPF programs to know the ztunnel IP on the current node

## Solution

In version 0.7.2, cgroup id is used to improve the performance of the connect program.
Usually, each container in a Pod has a proper cgroup id, which can be obtained through the `bpf_get_current_cgroup_id`
function in the BPF program. The speed of the connect program can be optimized by writing IP information to a specific `cgroup_info_map`.

An ambient mesh is different from the legacy CNI listening on a special port in the network namespace for storing Pod-related information.
In the ambient mesh, cgroup id is useful. If cgroup id can be associated with the Pod IP, you can get the current Pod IP in eBPF.

Since CNI cannot be relied on anymore, we need change the scheme for obtaining the information of Pod status.
For this reason, we detect the creation and revocation actions of local Pods by watching the process creation and revocation.
We created a new tool to watch the process changes on a host: [process-watcher project](https://github.com/merbridge/process-watcher).

Read the cgroup id and ip information from the process ID and writing it to the `cgroup_info_map`.

```go
tcg := cgroupInfo{
		ID:            cgroupInode,
		IsInMesh:      in,
		CgroupIp:      *(*[4]uint32)(_ip),
		Flags:         flag,
		DetectedFlags: cgrinfo.DetectedFlags | AMBIENT_MESH_MODE_FLAG | ZTUNNEL_FLAG,
	}
	return ebpfs.GetCgroupInfoMap().Update(&cgroupInode, &tcg, ebpf.UpdateAny)
```

Then get the current cgroup-related information in eBPF:

```c
__u64 cgroup_id = bpf_get_current_cgroup_id();
void *info = bpf_map_lookup_elem(&cgroup_info_map, &cgroup_id);
```

Now, we can learn whether the current container has the ambient mesh enabled and it is located in a mesh or not.

Second, for the ztunnel IP, Istio implements it by adding NIC and binding fixed IPs.
This scheme may have the risk of conflict, and the original addresses may be lost in some cases (such as SNAT).
So Merbridge gives up the scheme and directly obtains the ztunnel IPs on the control plane,
writes it into the map, and enables the eBPF program read it (this is faster).

```c
static inline __u32 *get_ztunnel_ip()
{
    __u32 ztunnel_ip_key = ZTUNNEL_KEY;
    return (__u32 *)bpf_map_lookup_elem(&settings, &ztunnel_ip_key);
}
```

Then use the connect program to rewrite the destination address:

```c
ctx->user_ip4 = ztunnel_ip[3];
ctx->user_port = bpf_htons(OUT_REDIRECT_PORT);
```

With the association with the cgroup id, the Pod IP of current processes can be obtained in eBPF, so as to enforce policies.
Forward the traffic from the Pod in the ambient mesh to the ztunnel, so that Merbridge can be compatible with the ambient mesh.

This will be a capability that is adaptable to all CNIs and can avoid the problem that the native ambient mesh cannot work well in most CNI modes.

## Usage and feedback

Since the ambient mesh is still in its early stage and the support for ambient mode is relatively preliminary,
some problems have not been well resolved, so the code of supporting for the ambient mode has not been merged into the main branch.
If you want to experience the capability of Merbridge to implement traffic interception for ambient mesh instead of iptables,
you can perform the following steps (it is required to install the ambient mesh in advance):

1. Disable Istio CNI (set `--set components.cni.enabled=false` during installation, or delete Istio CNI's DaemonSet `kubectl -n istio-system delete ds istio-cni`).
2. Remove the init container of ztunnel (because it initializes iptables rules and NICs, which is not required for Merbridge).
3. Install Merbridge by running `kubectl apply -f https://github.com/merbridge/merbridge/raw/ambient/deploy/all-in-one.yaml`

After the Merbridge is ready, you can use all capabilities of ambient mesh.

**\*Attentions:**

1. The Ambient mode under Kind is not supported currently (we have a plan to support it in the future)
2. The host kernel version needs to be not less than 5.7
3. cgroup v2 is required to be enabled
4. This mode is also compatible with sidecars
5. The debug mode will be enabled by default in an ambient mesh, which will have certain impact on performance

For more details see [source code](https://github.com/merbridge/merbridge/tree/ambient).

If you have any question, please reach out to us with [Slack](https://join.slack.com/t/merbridge/shared_invite/zt-11uc3z0w7-DMyv42eQ6s5YUxO5mZ5hwQ) or add the wechat group to chat.
