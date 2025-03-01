---
title: "Overview of Kubernetes CNI Network Models: VETH & Bridge / Overlay / BGP"
source: https://medium.com/@rifewang/overview-of-kubernetes-cni-network-models-veth-bridge-overlay-bgp-ea9bfa621d32
clipped: 2025-03-01
published: 
category: network
tags:
  - k8s
read: false
---


Networking is the foundation of communication between containers, and Kubernetes does not provide out-of-the-box network connectivity. Instead, it imposes two basic requirements:

-   Pods can communicate with all other pods on any other node without NAT.
-   Agents on a node (e.g., system daemons, kubelet) can communicate with all pods on that node.

The implementation of these networking capabilities usually relies on `CNI` plugins.

Kubernetes’ network model requires each Pod to have a unique IP address. The responsibility of managing and allocating these IP addresses falls to `IPAM` (IP Address Management), which is an important part of `CNI` plugins.

A common `IPAM` implementation assigns a CIDR (Classless Inter-Domain Routing) to each node and then allocates Pod IP addresses within that CIDR.

For example, setting the CIDR for each node:

```
apiVersion: v1  
kind: Node  
metadata:  
  name: node01  
spec:  
  podCIDR: 192.168.1.0/24  
  podCIDRs:  
  \- 192.168.1.0/24

\---  
  
apiVersion: v1  
kind: Node  
metadata:  
  name: node02  
spec:  
  podCIDR: 192.168.2.0/24  
  podCIDRs:  
  \- 192.168.2.0/24
```

In this example:

-   The `podCIDR` for node01 is `192.168.1.0/24` (address range: 192.168.1.0 ~ 192.168.1.255), so the Pod IP range for this node is `192.168.1.1 ~ 192.168.1.254` (the first and last addresses are reserved for other purposes).
-   The `podCIDR` for node02 is `192.168.2.0/24` (address range: 192.168.2.0 ~ 192.168.2.255), so the Pod IP range for this node is `192.168.2.1 ~ 192.168.2.254`.

This essentially assigns a small subnet to each node, where all Pod IPs on the same node are within the same subnet.

Note! This is just one possible implementation of `IPAM`. Different `CNI` plugins may offer other implementations.

Once the Pod IP addresses are assigned, the next challenge is how to enable communication within the cluster.

Linux provides a variety of virtual network interface types to support complex networking environments. Among these, `VETH` (virtual Ethernet) and `Bridge` are two key types.

![[Raw/Media/Resources/15318e129cb7f52b06d6239595760d64_MD5.png]]

As shown in the diagram above (you may have seen similar images before), each Pod has its own network namespace, which connects to the root namespace (the host network namespace) via a veth pair. A bridge (often named `cni0` or `docker0`) is then used to connect all veth pairs together.

It’s important to note that `VETH` and `Bridge` are independent technologies. If we use a veth pair to directly connect two pods, those pods can communicate with each other. However, using only veth pairs would result in an unmanageable number of pairs as the pod count grows, which is one reason why a bridge is employed.

Communication between pods on the same node is straightforward; the veth pair and bridge handle it locally.

But what happens when communication crosses nodes? We might assume the bridge would simply forward traffic to the root namespace’s eth0 physical network interface. Is it really that simple?

Of course not! Nodes may be virtual or physical machines, connected either via virtual networking or physical routers and switches. The critical question is: **Can the network device connecting the nodes (whether virtual or physical routers/switches/other devices) route Pod MAC/IP addresses directly?** (Routing node IPs is natural, but what about routing Pod IPs within nodes?)

If the answer is yes, then the `CNI` plugin can use just `VETH` and `Bridge` to achieve full container networking within the cluster. This is the first `CNI` network model discussed in this article. In `Cilium`, this model is called `Native-Routing`, while in `Flannel`, it is known as the `host-gw` mode.

As mentioned, if the routing device between nodes cannot route Pod IP addresses, how can we ensure cluster-wide communication? The answer is to use an `Overlay` network.

Since nodes can communicate (node IPs are routable), the original packet is encapsulated before transmission, such as with `VXLAN` encapsulation:

![[Raw/Media/Resources/1698b70962c0c42c44989d9a3979e163_MD5.jpg]]

As shown, the original Pod IP address is encapsulated inside the Inner Ethernet Frame, with the node IP address added to the outer layer. Since node IPs are routable, the packet is forwarded to the target node, where it is decapsulated and forwarded to the correct Pod based on the inner Pod IP.

`Overlay` networking allows Pods to communicate across nodes by encapsulating additional layers of networking on top of the existing network.

`Overlay` can be implemented in various ways, with `VXLAN` being a common approach. Other methods include `IP-in-IP`, among others. Linux provides direct support for `VXLAN` within the kernel, though different `CNI` plugins may implement their own `VXLAN` encapsulation and decapsulation methods outside the kernel.

This is the second `CNI` network model discussed in this article: `Overlay`.

In multi-cluster or hybrid cloud scenarios, there are cases where Pods within a cluster need to be accessible externally. In these situations, `BGP` (Border Gateway Protocol) is one solution.

`BGP` is widely used for routing in large-scale data centers or backbone networks. It allows different AS (autonomous systems) to exchange routing information.

![[Raw/Media/Resources/993f9a8fdde28efda1050647c21dd882_MD5.jpg]]

In Kubernetes, a cluster can be treated as an AS, and `BGP` can be used to exchange routing information. This allows Pods to be directly routable from outside the cluster, ensuring their external accessibility.

Given that `BGP` can exchange routing information, is it feasible to use `BGP` to exchange all Pod IP addresses within the cluster, allowing us to use the first `CNI` model (`Native-Routing`) directly? While feasible, it is not recommended due to the lack of programmability in the data path with `BGP`.

This article introduced three `CNI` network models: `Native-Routing`, `Overlay`, and `BGP`. The prerequisite for `Native-Routing` is that Pod IPs within the cluster must be routable, but this is often not the case. Therefore, many `CNI` plugins implement `Overlay` networks using techniques like `VXLAN` or `IP-in-IP` to ensure connectivity. In more complex scenarios, such as multi-cluster or hybrid cloud environments, external access to Pods can be achieved using `BGP`.