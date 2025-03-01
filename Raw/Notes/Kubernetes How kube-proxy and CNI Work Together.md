---
title: "Kubernetes: How kube-proxy and CNI Work Together"
source: https://medium.com/@rifewang/kubernetes-how-kube-proxy-and-cni-work-together-1255d273f291
clipped: 2025-03-01
published: 
category: network
tags:
  - k8s
read: false
---


In Kubernetes, `kube-proxy` and `CNI` plugins collaborate to ensure seamless communication between Pods within the cluster.

![[Raw/Media/Resources/6a9f7118bad26e6e62b90994fa157e08_MD5.png]]

As shown in the image above, let’s assume we have a `Service` of type `ClusterIP` that corresponds to two Pods located on different nodes.

When Pod A initiates a request to this `Service`:

1.  `Pod A: 192.168.0.2 --> service-name` (accessing the Service via domain name).
2.  CoreDNS resolves the domain name and returns the `ClusterIP` address for the service: `Pod A: 192.168.0.2 --> 10.111.13.31`
3.  The Linux kernel’s Netfilter performs DNAT (Destination Network Address Translation) and selects a real backend Pod IP: `Pod A: 192.168.0.2 --> 192.168.1.4`
4.  If the requested Pod resides on a different node, the CNI plugin, based on its network configuration, either:

-   Directly forwards the packet via the host network, or
-   Encapsulates the packet using VXLAN/IPIP and then forwards it.

This is the basic flow, but more details come into play in actual practice.

`kube-proxy` is typically deployed as a DaemonSet to ensure each worker node has a running pod. It connects to the `kube-apiserver`, listens for Service objects (and related Endpoints or EndpointSlices), and then configures the node’s `Netfilter` using either `iptables` or `ipvs`.

`Netfilter` is a framework within the Linux kernel that processes network packets. It operates at several points in the network protocol stack (e.g., input, output, forwarding), allowing for packet filtering, modification, and redirection. Its hook system enables inserting custom rules at different stages of packet processing.

![[Raw/Media/Resources/6092656870f1afa8aa4c5f9468c9f883_MD5.png]]

Both `iptables` and `ipvs` utilize the functionalities of `Netfilter`.

A Service’s `ClusterIP` is a virtual IP (`VIP`), meaning it doesn’t have a physical network entity but is used for handling packet routing.

`kube-proxy` uses `iptables` or `ipvs` to configure Service `ClusterIP` (VIP) and DNAT/SNAT rules. When Pod A accesses the `ClusterIP`, the Linux kernel replaces the virtual IP with a real Pod IP based on these rules.

Viewing the `iptables` rules might look like this:

![[Raw/Media/Resources/83e5625f54044de934773d02e2d25abf_MD5.png]]

And `ipvs` rules might appear like this:

![[Raw/Media/Resources/3db500817cc4e5428bfa02e17fc39139_MD5.png]]

A `Service` can correspond to multiple Pods, bringing in load-balancing concerns. `iptables` uses random selection, while `ipvs` supports various algorithms like `rr` (Round Robin, default), `lc` (Least Connection), `sh` (Source Hashing), and more.

In short, `kube-proxy` listens for Services and uses `iptables` or `ipvs` to configure the Linux kernel’s packet modification and forwarding rules.

After the Linux kernel completes DNAT/SNAT, the next step is to send the packet. If the Pod’s MAC/IP is reachable within the cluster network, the CNI plugin doesn’t need to perform additional actions and can directly forward the packet via the host network. However, often this condition is unmet, and the CNI plugin provides an `Overlay` network mode, encapsulating the packet using VXLAN or IPIP before transmission.

For more details on the CNI network model, you can refer to my previous article *“Kubernetes CNI Network Overview: VETH & Bridge / Overlay / BGP.”*

Common CNI plugins like `Flannel` and `Calico` use VXLAN or IPIP based on Linux kernel capabilities. The CNI plugin's primary role is configuration.

This means both `kube-proxy` and the CNI plugin configure the Linux kernel, and the kernel handles packet processing. Some CNI plugins, like `Cilium`, leverage technologies like `eBPF` to bypass the kernel and process packets directly in userspace.

`kube-proxy` has some limitations. For example, as the number of Services and Pods grows in a cluster, the efficiency of `iptables` rule matching decreases. Even with optimizations in `ipvs`, performance overhead still exists. Frequent updates to Services and Pods trigger rule reapplications, potentially leading to network delays or disruptions. Additionally, `kube-proxy` relies on the Linux kernel's `Netfilter`, and misconfigurations or incompatibilities in the kernel can cause network issues.

In fact, `kube-proxy` performs relatively simple tasks. If the network plugin can achieve efficient packet forwarding and provide equivalent functionality, `kube-proxy` may no longer be necessary. For example, `Cilium` uses `eBPF` to implement a proxy-free service traffic forwarding mechanism, making `kube-proxy` redundant.

In Kubernetes clusters, `kube-proxy` and `CNI` plugins work together by configuring the Linux kernel’s networking components (such as `Netfilter`, `VXLAN`, etc.) to ensure communication between Pods and Services. With advancements in technologies like `eBPF`, some CNI plugins now enable proxy-free service forwarding, further optimizing network performance.