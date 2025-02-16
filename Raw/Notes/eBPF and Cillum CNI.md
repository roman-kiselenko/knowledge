---
title: eBPF and Cillum CNI
source: https://sigridjin.medium.com/ebpf-and-cillum-cni-1c3ad2a1e4e9
clipped: 2025-02-16
published: 
category: network
tags:
  - network
  - cni
read: false
---

## Cilium CNI, the way of Advanced Networking and Security for Kubernetes with eBPF

## Introducing eBPF

eBPF (Extended Berkeley Packet Filter) represents a revolutionary technology that allows programs to run within the Linux kernel without modifying kernel source code or loading kernel modules. While its name suggests a focus on packet filtering, eBPF has evolved far beyond its original networking roots to become a powerful and versatile framework for developing kernel-space applications.

![[Raw/Media/Resources/4feddf4b0b764daf6a49d45bcfdbcd3d_MD5.png]]

At its core, eBPF enables the execution of sandboxed programs in privileged kernel contexts, providing unprecedented abilities for networking, observability, and security use cases. Think of it as a “mini-VM” inside the Linux kernel that can safely execute programs at nearly native performance.

A sandbox program, in the context of eBPF, operates in an isolated environment with strict security guarantees. The eBPF verifier ensures that these programs…

1.  Cannot crash the kernel
2.  Always terminate (no infinite loops)
3.  Can only access authorized memory
4.  Follow strict security policies This makes eBPF fundamentally different from traditional kernel modules, offering safety without sacrificing performance.

![[Raw/Media/Resources/a732254528c20194f0d767de3d3ac8ae_MD5.png]]

The conventional Linux networking stack, while robust, presents several significant limitations.

**Complexity**. The traditional stack follows a rigid layered approach:

-   Layer 1: Hardware layer (NICs and drivers)
-   Layer 2: Ethernet and data link layer
-   Layer 3: IP and Netfilter subsystem
-   Layer 4: TCP/UDP transport
-   Layer 5: Session layer with socket operations
-   Layer 7: Application layer with system call interface

![[Raw/Media/Resources/6935b645fcbbcf06b18b402334b2a2ee_MD5.png]]

**Performance Overhead**. Consider a typical network operation where a user-space application communicates with external systems. The data path involves…

-   Application initiates `sendmsg()` system call
-   Traversal through socket layer
-   Processing through user-space networking
-   Transition to kernel network device
-   Finally reaching external networks This process repeats in reverse for receiving data `recvmsg()`, creating significant overhead.

The limitations of iptables, the traditional Linux firewall system, have become increasingly apparent in modern, containerized environments.

**Rule Management Complexity**

-   Rules must be recreated and updated as complete transactions
-   The linked-list implementation of rule chains results in O(n) complexity
-   Each packet must traverse the entire rule chain from the beginning
-   Performance degradation becomes severe as rule sets grow

**Scalability Issues**

-   Large rule sets significantly impact system performance
-   Rule updates require atomic operations, making dynamic environments challenging
-   Container environments often require frequent rule updates, exacerbating these issues

![[Raw/Media/Resources/30f9ce1dbc3edbeeae4e333da20e7c3d_MD5.png]]

Here is the place where eBPF comes in, which introduces a more flexible and efficient approach through kernel hooks. These hooks can be inserted at various points in the kernel, allowing for the following.

-   Direct packet inspection and manipulation
-   Custom processing logic at multiple stages
-   Efficient policy enforcement
-   Dynamic tracing and monitoring
-   Performance optimization through bypass mechanisms

The ability to place hooks at almost any point in the kernel execution path makes eBPF particularly powerful for modern networking requirements, especially in cloud-native environments where traditional networking paradigms often fall short.

This enhanced flexibility and performance have made eBPF the foundation for many modern networking tools and platforms, such as Cilium for container networking and security, and various observability solutions that provide deep insights into system behavior.

BPF’s kernel hooks represent a fundamental advancement in Linux system programming, offering the ability to insert control points directly into the kernel for packet filtering and system monitoring. This capability extends across various kernel subsystems, providing unprecedented access and control over system operations. The flexibility to insert hooks at virtually any point transforms the traditionally static kernel behavior into a dynamic, programmable environment.

The evolution of this technology tells an interesting story. Starting with the original Berkeley Packet Filter (BPF) in 1992, the technology made a quantum leap with the introduction of eBPF in 2014. This extension dramatically broadened the scope of applications, enabling adoption across security frameworks, system tracing, advanced networking, performance monitoring, and observability tools. The timing of this evolution coincided perfectly with the rising needs of cloud computing and containerized environments.

The technical foundation of eBPF centers on its unique ability to execute sandboxed programs within privileged kernel contexts. This architectural approach delivers two critical advantages: the ability to run programs at the kernel level with robust safety guarantees, and the flexibility to modify kernel behavior without directly altering kernel code. This safety-first approach has been crucial for production adoption.

![[Raw/Media/Resources/299e1df974d1d734bc4d1d826df831d1_MD5.png]]

Looking at the hook architecture, eBPF implements an event-driven model where programs execute in response to specific kernel or application triggers at predefined hook points. These triggers can include system calls, function entry and exit points, kernel tracepoints, and network events. This hook system enables dynamic instrumentation, real-time monitoring, programmatic control flow modification, and zero-copy data access, all critical capabilities for modern system observability and networking solutions.

The native integration of eBPF into the Linux kernel provides several significant architectural benefits. First, performance optimization is achieved through direct kernel-space processing, bypassing user-space transitions, reducing context switching overhead, and enabling zero-copy operations for network data. Second, safety and reliability are ensured through a verification system that prevents system crashes, validates memory access, and guarantees bounded execution. Third, runtime efficiency is maximized through JIT compilation support, native instruction execution, and minimal overhead compared to traditional kernel modules.

![[Raw/Media/Resources/f63e08ff510f703b928a987d5c81a032_MD5.png]]

The technical implementation incorporates several sophisticated mechanisms. The verification process ensures program safety without compromising performance, while JIT compilation converts eBPF bytecode to native machine code for optimal execution. The direct kernel integration enables microsecond-level response times, and hook points can be dynamically manipulated without system restarts. These capabilities make eBPF particularly valuable for high-performance networking applications, security monitoring and enforcement, system observability tools, custom kernel extensions, and container networking solutions.

From an engineering perspective, eBPF represents a perfect balance between safety and performance. The architecture combines kernel-level execution capabilities with comprehensive safety checks and efficient JIT compilation, positioning it uniquely for modern infrastructure requirements. This becomes especially important in cloud-native environments and high-performance computing scenarios where traditional approaches often fall short.

The implications for system designers and developers are significant. eBPF provides a way to implement complex networking, security, and monitoring solutions with minimal overhead and maximum flexibility. The ability to dynamically insert and modify kernel behavior without reboots or module loading makes it particularly suitable for production environments where downtime is not acceptable.

![[Raw/Media/Resources/0d32cd9ddc50f837987cf767d6a2f8c3_MD5.png]]

In typical network operations, packets follow a predetermined path through various kernel layers. This traditional approach, while functional, isn’t always optimal for high-performance requirements. Let’s examine how different packet processing methods affect performance, particularly in high-throughput scenarios.

![[Raw/Media/Resources/ac228cb5aa7c1a4856de81998ead3636_MD5.png]]

Let’s say that you conduct packet processing using a 10GbE network interface. The key metric we’re focusing on is the relationship between packet dropping and processing speed. This relationship is crucial because the system’s overall performance depends on how quickly packets can be processed or dropped, especially under high load conditions.

When packets are handled at the userspace layer, they follow a complex path.

-   hardware ingress → TC Ingress → Iptables rules → Application layer.

![[Raw/Media/Resources/4f5d0998961d4a3de22060af33675dc0_MD5.png]]

This creates multiple context switches and copying operations, significantly impacting performance.

![[Raw/Media/Resources/b0a9f6085fcc95498b7202eec2882f58_MD5.png]]

With Netfilter-based packet handling, the path is shortened: hardware ingress → TC Ingress → Iptables rules. While more efficient than userspace processing, it still involves several kernel subsystems.

![[Raw/Media/Resources/787690589d6fe36dde50504cd782e3e1_MD5.png]]

Traffic Control (TC) Ingress processing provides a more direct path: hardware ingress → L3 TC Ingress. This reduces the processing overhead by intercepting packets earlier in the network stack.

![[Raw/Media/Resources/6543fdb1dcdf48bf73c8834ee40c86f2_MD5.png]]

XDP represents the most efficient approach, operating at the network driver level before packets enter the main kernel network stack.

Performance Comparison Results:  
\- Userspace Processing: 783,063 packets/second  
\- Netfilter Processing: 1,266,730 packets/second  
\- TC Ingress Processing: 4,083,820 packets/second  
\- XDP Processing: 6.69 Gbps throughput

XDP achieves significantly higher throughput because it operates at the network driver level, allowing packet processing or dropping before they traverse higher layers like userspace, Netfilter, or TC.

![[Raw/Media/Resources/ac35b0ec5ff2918bf44a3c7519ed50a7_MD5.png]]

This early interception dramatically reduces latency and improves overall system performance. In contrast, processing packets through Netfilter or TC requires more system resources and involves additional layers of the network stack, resulting in lower performance.

![[Raw/Media/Resources/43042e54029fa93c6cc8b5842f1f3ad0_MD5.png]]

[https://lakescript.net/entry/KANS-Cilium-CNI-eBPF](https://lakescript.net/entry/KANS-Cilium-CNI-eBPF)

Similar to iptables, XDP uses rules for packet processing. However, its key advantage is early packet interception. When packets must traverse to netfilter or higher layers, they incur additional processing overhead, reducing performance.

![[Raw/Media/Resources/e8281ce2122d3c8549a5cc5565127f23_MD5.png]]

-   Generic Mode: Operates within the Linux Kernel Network Stack
-   Native Mode: Runs at the Network Driver level (e.g., Intel drivers)
-   Offloaded Mode: Executes directly on Network Hardware (e.g., Netronome cards)

![[Raw/Media/Resources/a4f1f03774c249b6b73ebecbdded256f_MD5.png]]

When configured in offload mode, XDP achieves maximum efficiency by processing packets directly at the hardware level. This configuration can handle the full 10Gbps throughput, as packet dropping occurs before any kernel involvement, eliminating software processing overhead entirely.

The performance advantages of XDP become particularly evident in high-throughput scenarios where traditional packet processing methods become bottlenecks. This makes it especially valuable for applications requiring high-performance networking, such as DDoS mitigation, load balancing, and network monitoring at scale.

## Introducing Cilium CNI

Cilium CNI represents a significant advancement in container networking, serving as an open-source networking plugin specifically designed for container orchestration platforms like Kubernetes. Its distinguishing feature lies in its use of eBPF (extended Berkeley Packet Filter) technology, enabling high-performance network packet processing directly at the kernel level.

![[Raw/Media/Resources/388c001546459156808a7ec9ac6320cc_MD5.png]]

At its core, Cilium functions as a CNI Plugin that leverages eBPF to provide both Pod networking capabilities and security features. The implementation is particularly elegant in its approach to packet handling: Cilium attaches eBPF programs to the Traffic Control (TC) ingress hooks of network interfaces, allowing it to intercept and process all incoming packets with remarkable efficiency.

![[Raw/Media/Resources/b6e62f6b51d6cea11c977b19511bd235_MD5.png]]

Cilium CNI offers two distinct networking modes to accommodate different deployment scenarios: Tunnel Mode and Native-Routing Mode.

The Tunnel Mode operates by implementing either VXLAN or Geneve tunneling protocols. In VXLAN configuration, it utilizes UDP port 8472, while Geneve operates on UDP port 6081. These protocols establish virtual tunnels that enable network traffic flow between pods across different networks. This approach effectively creates a virtual overlay network, facilitating seamless pod-to-pod communication regardless of their physical location within the cluster.

In contrast, Native-Routing Mode takes a different approach by utilizing the network’s inherent routing capabilities without tunneling. This mode requires external routing configuration for pod-to-pod communication, particularly for traffic crossing cluster boundaries. It proves particularly beneficial in cloud environments where reducing traffic overhead is crucial, as it eliminates the additional encapsulation layer present in tunnel mode.

![[Raw/Media/Resources/0a2b7641bc4ae117ff6a8a8b767d3962_MD5.png]]

One of Cilium’s most significant technical achievements is its ability to operate with minimal reliance on iptables. Through its eBPF implementation, Cilium handles Masquerading (SNAT) processing directly, though some edge cases involving iptables functionality continue to be addressed through ongoing development efforts.

![[Raw/Media/Resources/0a4a807db328a59d31f6b184a8a5bb17_MD5.png]]

[https://docs.cilium.io/en/stable/overview/component-overview/](https://docs.cilium.io/en/stable/overview/component-overview/)

The Cilium architecture comprises several key components working in concert to manage networking and security in Kubernetes clusters.

The Cilium Agent operates as a DaemonSet on each node, managing everything from Kubernetes API configurations to network settings, policy enforcement, load balancing, and monitoring. It maintains control over eBPF programs to regulate pod traffic effectively.

Cilium Client (CLI) provides direct access to eBPF maps for state inspection and management. The Cilium Operator handles cluster-wide IP allocation and maintains consistency across various cluster operations, including IP address initialization and inter-node synchronization.

![[Raw/Media/Resources/638973b6a4fd5d63f312389989953200_MD5.png]]

Hubble serves as a distributed networking and security observability platform built atop Cilium and eBPF. Its implementation provides deep visibility into service communications and network infrastructure operations without requiring application modifications.

Hubble’s architecture includes several sophisticated capabilities:

-   Comprehensive network and security monitoring across containerized workloads
-   Support for virtual machines and traditional Linux processes
-   Service, pod, and identity-based monitoring and control
-   Application layer filtering capabilities, including HTTP traffic
-   Integration with Prometheus for metrics export

![[Raw/Media/Resources/eec4bf11c890c4818daff9d25ed7d910_MD5.png]]

Cilium introduces an innovative approach to load balancing through its socket-based implementation. Traditional network-based load balancing requires DNAT transformation through host iptables when frontend pods communicate with ClusterIP services. Cilium’s socket-based approach transforms Service IP to backend Pod IP directly during the `connect()` system call, eliminating the need for intermediate DNAT conversions.

This implementation significantly improves performance by reducing networking overhead and simplifying the packet routing process. The direct transformation at the socket level ensures that all subsequent packets are automatically directed to the correct backend address without additional translation steps.

## Practices

> Environment Requirements
> 
> VPC with 2 public subnets  
> 3 EC2 instances (Ubuntu 22.04 LTS, t3.xlarge with 4 vCPU and 16GB Memory)  
> 1 test instance (t3.small)

Cilium Installation Process

1.  Adding Helm Repository: First, add the Cilium repository to Helm and update: helm repo add cilium [https://helm.cilium.io/](https://helm.cilium.io/) helm repo update
2.  Cilium Installation via Helm: Execute the comprehensive Helm installation with specific configuration parameters:

The installation command includes crucial parameters for optimal performance and functionality:

Key Configuration Parameters Explained:

-   debug.enabled: Enables debug-level logging in Cilium pods
-   autoDirectNodeRoutes: Enables automatic routing configuration between nodes in the same network range for podCIDR ranges
-   endpointRoutes.enabled: Configures individual routing for each endpoint (pod) on the host
-   hubble.relay.enabled and hubble.ui.enabled: Activates Hubble monitoring capabilities
-   ipam.mode=kubernetes: Utilizes Kubernetes IPAM
-   kubeProxyReplacement: Maximizes kube-proxy replacement capabilities
-   ipv4NativeRoutingCIDR: Specifies the network range that doesn’t require IP masquerading
-   installNoConntrackIptablesRules: Disables conntrack usage in iptables rules
-   bpf.masquerade: Implements masquerading through BPF instead of iptables

![[Raw/Media/Resources/1506b926ae5be598ee66a9cee767bb15_MD5.png]]

helm repo add cilium https://helm.cilium.io/  
helm repo update

helm install cilium cilium/cilium 

(⎈|kubernetes-admin@kubernetes:N/A) root@k8s-s:~# ip -c addr  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
    inet 127.0.0.1/8 scope host lo  
       valid\_lft forever preferred\_lft forever  
    inet6 ::1/128 scope host  
       valid\_lft forever preferred\_lft forever  
2: ens5: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 9001 qdisc mq state UP group default qlen 1000  
    link/ether 02:f2:e6:5d:bf:8b brd ff:ff:ff:ff:ff:ff  
    altname enp0s5  
    inet 192.168.10.10/24 metric 100 brd 192.168.10.255 scope global dynamic ens5  
       valid\_lft 2937sec preferred\_lft 2937sec  
    inet6 fe80::f2:e6ff:fe5d:bf8b/64 scope link  
       valid\_lft forever preferred\_lft forever  
3: cilium\_net@cilium\_host: <BROADCAST,MULTICAST,NOARP,UP,LOWER\_UP> mtu 9001 qdisc noqueue state UP group default qlen 1000  
    link/ether 56:1b:05:38:ec:fd brd ff:ff:ff:ff:ff:ff  
    inet6 fe80::541b:5ff:fe38:ecfd/64 scope link  
       valid\_lft forever preferred\_lft forever  
4: cilium\_host@cilium\_net: <BROADCAST,MULTICAST,NOARP,UP,LOWER\_UP> mtu 9001 qdisc noqueue state UP group default qlen 1000  
    link/ether d6:90:dc:cf:24:9f brd ff:ff:ff:ff:ff:ff  
    inet 172.16.0.84/32 scope global cilium\_host  
       valid\_lft forever preferred\_lft forever  
    inet6 fe80::d490:dcff:fecf:249f/64 scope link  
       valid\_lft forever preferred\_lft forever  
6: lxc\_health@if5: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 9001 qdisc noqueue state UP group default qlen 1000  
    link/ether a2:ae:f5:13:75:c0 brd ff:ff:ff:ff:ff:ff link-netnsid 0  
    inet6 fe80::a0ae:f5ff:fe13:75c0/64 scope link  
       valid\_lft forever preferred\_lft forever  
8: lxcb8e3b22d7b27@if7: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 9001 qdisc noqueue state UP group default qlen 1000  
    link/ether 9a:58:74:2f:f1:ab brd ff:ff:ff:ff:ff:ff link-netns cni-243caf18-3769\-b2b8-ab46-d9ab7b37d502  
    inet6 fe80::9858:74ff:fe2f:f1ab/64 scope link  
       valid\_lft forever preferred\_lft forever  
10: lxcb76fcb8847b3@if9: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 9001 qdisc noqueue state UP group default qlen 1000  
    link/ether b2:6d:f1:73:a6:6c brd ff:ff:ff:ff:ff:ff link-netns cni-95ffa813-f06e-a47b-02a2-98e48b7ec5b1  
    inet6 fe80::b06d:f1ff:fe73:a66c/64 scope link  
       valid\_lft forever preferred\_lft forever

$ kubectl get node,pod,svc -A -owide

NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE     IP               NODE     NOMINATED NODE   READINESS GATES  
kube-system   pod/cilium-9rs9r                       1/1     Running   0          69s     192.168.10.10    k8s-s    <none\>           <none\>  
kube-system   pod/cilium-envoy-9f2gq                 1/1     Running   0          69s     192.168.10.10    k8s-s    <none\>           <none\>  
kube-system   pod/cilium-envoy-hrsj7                 1/1     Running   0          69s     192.168.10.101   k8s-w1   <none\>           <none\>  
kube-system   pod/cilium-envoy-vf5dg                 1/1     Running   0          69s     192.168.10.102   k8s-w2   <none\>           <none\>  
kube-system   pod/cilium-hngxs                       1/1     Running   0          68s     192.168.10.102   k8s-w2   <none\>           <none\>  
kube-system   pod/cilium-operator-76bb588dbc-sgtdm   1/1     Running   0          69s     192.168.10.102   k8s-w2   <none\>           <none\>  
kube-system   pod/cilium-xlstd                       1/1     Running   0          68s     192.168.10.101   k8s-w1   <none\>           <none\>  
kube-system   pod/coredns-55cb58b774-qqpkh           1/1     Running   0          9m39s   172.16.0.51      k8s-s    <none\>           <none\>  
kube-system   pod/coredns-55cb58b774-tlcnd           1/1     Running   0          9m39s   172.16.0.103     k8s-s    <none\>           <none\>  
kube-system   pod/etcd-k8s-s                         1/1     Running   0          9m52s   192.168.10.10    k8s-s    <none\>           <none\>  
kube-system   pod/hubble-relay-88f7f89d4-sfnv6       1/1     Running   0          69s     172.16.2.156     k8s-w1   <none\>           <none\>  
kube-system   pod/hubble-ui-59bb4cb67b-dsqbv         2/2     Running   0          69s     172.16.2.127     k8s-w1   <none\>           <none\>  
kube-system   pod/kube-apiserver-k8s-s               1/1     Running   0          9m52s   192.168.10.10    k8s-s    <none\>           <none\>  
kube-system   pod/kube-controller-manager-k8s-s      1/1     Running   0          9m52s   192.168.10.10    k8s-s    <none\>           <none\>  
kube-system   pod/kube-scheduler-k8s-s               1/1     Running   0          9m52s   192.168.10.10    k8s-s    <none\>           <none\>

NAMESPACE     NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE     SELECTOR  
default       service/kubernetes       ClusterIP   10.10.0.1       <none\>        443/TCP                  9m54s   <none\>  
kube-system   service/cilium-envoy     ClusterIP   None            <none\>        9964/TCP                 69s     k8s-app=cilium-envoy  
kube-system   service/hubble-metrics   ClusterIP   None            <none\>        9965/TCP                 69s     k8s-app=cilium  
kube-system   service/hubble-peer      ClusterIP   10.10.162.103   <none\>        443/TCP                  69s     k8s-app=cilium  
kube-system   service/hubble-relay     ClusterIP   10.10.7.17      <none\>        80/TCP                   69s     k8s-app=hubble-relay  
kube-system   service/hubble-ui        ClusterIP   10.10.26.235    <none\>        80/TCP                   69s     k8s-app=hubble-ui  
kube-system   service/kube-dns         ClusterIP   10.10.0.10      <none\>        53/UDP,53/TCP,9153/TCP   9m53s   k8s-app=kube-dns

(⎈|kubernetes\-admin@kubernetes:N/A) root@k8s\-s:~\# iptables \-t nat \-S  
\-P PREROUTING ACCEPT  
\-P INPUT ACCEPT  
\-P OUTPUT ACCEPT  
\-P POSTROUTING ACCEPT  
\-N CILIUM\_OUTPUT\_nat  
\-N CILIUM\_POST\_nat  
\-N CILIUM\_PRE\_nat  
\-N KUBE\-KUBELET\-CANARY  
\-A PREROUTING \-m comment \--comment "cilium-feeder: CILIUM\_PRE\_nat" \-j CILIUM\_PRE\_nat  
\-A OUTPUT \-m comment \--comment "cilium-feeder: CILIUM\_OUTPUT\_nat" \-j CILIUM\_OUTPUT\_nat  
\-A POSTROUTING \-m comment \--comment "cilium-feeder: CILIUM\_POST\_nat" \-j CILIUM\_POST\_nat

Verification and System Checks

1.  Network Interface Verification: ip -c addr This command displays detailed network interface information.
2.  Kubernetes Resource Verification: kubectl get node,pod,svc -A -owide Provides a comprehensive view of all Kubernetes resources across namespaces.
3.  NAT Configuration Check: iptables -t nat -S Examines the NAT table configurations, which should be notably clean due to Cilium’s eBPF implementation.
4.  Custom Resource Definition Verification: kubectl get crd Shows installed Custom Resource Definitions.
5.  Cilium Node Information: kubectl get ciliumnodes Displays information about nodes where Cilium agents are installed.

-   CILIUMINTERNALIP: Corresponds to cilium\_host interface IP
-   INTERNALIP: Node’s internal IP address

(⎈|kubernetes-admin@kubernetes:N/A) root@k8s-s:~# kubectl get crd  
NAME                                         CREATED AT  
ciliumcidrgroups.cilium.io                   2024-10-26T16:19:06Z  
ciliumclusterwidenetworkpolicies.cilium.io   2024-10-26T16:19:07Z  
ciliumendpoints.cilium.io                    2024-10-26T16:19:06Z  
ciliumexternalworkloads.cilium.io            2024-10-26T16:19:06Z  
ciliumidentities.cilium.io                   2024-10-26T16:19:06Z  
ciliuml2announcementpolicies.cilium.io       2024-10-26T16:19:06Z  
ciliumloadbalancerippools.cilium.io          2024-10-26T16:19:06Z  
ciliumnetworkpolicies.cilium.io              2024-10-26T16:19:07Z  
ciliumnodeconfigs.cilium.io                  2024-10-26T16:19:06Z  
ciliumnodes.cilium.io                        2024-10-26T16:19:06Z  
ciliumpodippools.cilium.io                   2024-10-26T16:19:06Z

Endpoint Verification: `kubectl get ciliumendpoints -A` Shows pod endpoints across all namespaces.

Network Driver Information: `ethtool -i ens5` Provides detailed information about network interface drivers, versions, and firmware.

Connection Tracking Configuration: `iptables -t raw -S | grep notrack` Verifies the NOTRACK rules implementation. This setting is crucial for performance optimization as it bypasses connection tracking for specific packets, reducing processing overhead.

(⎈|kubernetes\-admin@kubernetes:N/A) root@k8s\-s:~\# kubectl get ciliumnodes  
NAME     CILIUMINTERNALIP   INTERNALIP       AGE  
k8s\-s    172.16.0.84        192.168.10.10    78s  
k8s\-w1   172.16.2.121       192.168.10.101   77s  
k8s\-w2   172.16.1.95        192.168.10.102   65s

The NOTRACK configuration is particularly important for performance optimization. By bypassing the connection tracking system (conntrack) for certain packets, it reduces processing overhead and improves overall network performance. This is especially beneficial for high-throughput scenarios where connection state tracking isn’t necessary.

(⎈|kubernetes-admin@kubernetes:N/A) root@k8s-s:~  
NAMESPACE     NAME                           SECURITY IDENTITY   ENDPOINT STATE   IPV4           IPV6  
kube-system   coredns-55cb58b774-qqpkh       43458               ready            172.16.0.51  
kube-system   coredns-55cb58b774-tlcnd       43458               ready            172.16.0.103  
kube-system   hubble-relay-88f7f89d4-sfnv6   20419               ready            172.16.2.156  
kube-system   hubble-ui-59bb4cb67b-dsqbv     24934               ready            172.16.2.127

(⎈|kubernetes-admin@kubernetes:N/A) root@k8s-s:~# ethtool \-i ens5  
driver: ena  
version: 6.8.0\-1015\-aws  
firmware-version:  
expansion-rom-version:  
bus-info: 0000:00:05.0  
supports-statistics: yes  
supports-test: no  
supports-eeprom-access: no  
supports-register-dump: no  
supports-priv-flags: no

(⎈|kubernetes\-admin@kubernetes:N/A) root@k8s\-s:~\# iptables \-t raw \-S | grep notrack  
\-A CILIUM\_OUTPUT\_raw \-d 192.168.0.0/16 \-m comment \--comment "cilium: NOTRACK for pod traffic" \-j CT \--notrack  
\-A CILIUM\_OUTPUT\_raw \-s 192.168.0.0/16 \-m comment \--comment "cilium: NOTRACK for pod traffic" \-j CT \--notrack  
\-A CILIUM\_OUTPUT\_raw \-o lxc+ \-m comment \--comment "cilium: NOTRACK for proxy return traffic" \-j CT \--notrack  
\-A CILIUM\_OUTPUT\_raw \-o cilium\_host \-m comment \--comment "cilium: NOTRACK for proxy return traffic" \-j CT \--notrack  
\-A CILIUM\_OUTPUT\_raw \-o lxc+ \-m comment \--comment "cilium: NOTRACK for L7 proxy upstream traffic" \-j CT \--notrack  
\-A CILIUM\_OUTPUT\_raw \-o cilium\_host \-m comment \--comment "cilium: NOTRACK for L7 proxy upstream traffic" \-j CT \--notrack  
\-A CILIUM\_PRE\_raw \-d 192.168.0.0/16 \-m comment \--comment "cilium: NOTRACK for pod traffic" \-j CT \--notrack  
\-A CILIUM\_PRE\_raw \-s 192.168.0.0/16 \-m comment \--comment "cilium: NOTRACK for pod traffic" \-j CT \--notrack  
\-A CILIUM\_PRE\_raw \-m comment \--comment "cilium: NOTRACK for proxy traffic" \-j CT \--notrack

## Cilium CLI Installation and Configuration Guide

First, set up essential environment variables for the installation.

CILIUM\_CLI\_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)  
CLI\_ARCH=amd64

  
if \[ "$(uname -m)" = "aarch64" \]; then CLI\_ARCH=arm64; fi  
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM\_CLI\_VERSION}/cilium-linux-${CLI\_ARCH}.tar.gz{,.sha256sum}  
sha256sum --check cilium-linux-${CLI\_ARCH}.tar.gz.sha256sum  
sudo tar xzvfC cilium-linux-${CLI\_ARCH}.tar.gz /usr/local/bin  
rm cilium-linux-${CLI\_ARCH}.tar.gz{,.sha256sum}

![[Raw/Media/Resources/d36372ba8e2af81484317624a249acab_MD5.png]]

(⎈|kubernetes-admin@kubernetes:N/A) root@k8s-s:~# export CILIUMPOD0=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-s  -o jsonpath='{.items\[0\].metadata.name}')  
alias c0="kubectl exec -it $CILIUMPOD0 -n kube-system -c cilium-agent -- cilium"  
(⎈|kubernetes-admin@kubernetes:N/A) root@k8s-s:~# c0 status | grep KubeProxyReplacement  
KubeProxyReplacement:    True   \[ens5   192.168.10.10 fe80::f2:e6ff:fe5d:bf8b (Direct Routing)\]

This creates a convenient alias for accessing the Cilium agent container.

c0 status | grep KubeProxyReplacement

This checks KubeProxyReplacement configuration and confirms direct routing for the 192.168.0.0/16 network range without IP masquerading.

(⎈|kubernetes-admin@kubernetes:N/A) root@k8s\-s:~  
enable-bpf-masquerade                             true  
enable-ipv4-masquerade                            true  
enable-ipv6-masquerade                            true  
enable-masquerade-to-route-source                 false

It verifies the use of eBPF masquerade instead of iptables masquerade.

(⎈|kubernetes-admin@kubernetes:N/A) root@k8s-s:~# helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values --set ipMasqAgent.enabled=true  
Release "cilium" has been upgraded. Happy Helming!  
NAME: cilium  
LAST DEPLOYED: Sun Oct 27 01:24:19 2024  
NAMESPACE: kube-system  
STATUS: deployed  
REVISION: 2  
TEST SUITE: None  
NOTES:  
You have successfully installed Cilium with Hubble Relay and Hubble UI.

Your release version is 1.16.3.

For any further help, visit https://docs.cilium.io/en/v1.16/gettinghelp  
(⎈|kubernetes-admin@kubernetes:N/A) root@k8s-s:~# export CILIUMPOD0=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-s  -o jsonpath='{.items\[0\].metadata.name}')  
alias c0="kubectl exec -it $CILIUMPOD0 -n kube-system -c cilium-agent -- cilium"

It enables NAT for pod traffic leaving the cluster, protecting pod IPs from external exposure.

  
export CILIUMPOD0=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-s  -o jsonpath='{.items\[0\].metadata.name}')  
export CILIUMPOD1=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-w1 -o jsonpath='{.items\[0\].metadata.name}')  
export CILIUMPOD2=$(kubectl get -l k8s-app=cilium pods -n kube-system --field-selector spec.nodeName=k8s-w2 -o jsonpath='{.items\[0\].metadata.name}')

  
alias c0="kubectl exec -it $CILIUMPOD0 -n kube-system -c cilium-agent -- cilium"  
alias c1="kubectl exec -it $CILIUMPOD1 -n kube-system -c cilium-agent -- cilium"  
alias c2="kubectl exec -it $CILIUMPOD2 -n kube-system -c cilium-agent -- cilium"

alias c0bpf="kubectl exec -it $CILIUMPOD0 -n kube-system -c cilium-agent -- bpftool"  
alias c1bpf="kubectl exec -it $CILIUMPOD1 -n kube-system -c cilium-agent -- bpftool"  
alias c2bpf="kubectl exec -it $CILIUMPOD2 -n kube-system -c cilium-agent -- bpftool"

## Hubble UI Configuration

(⎈|kubernetes\-admin@kubernetes:N/A) root@k8s\-s:~\# kubectl patch \-n kube\-system svc hubble\-ui \-p '{"spec": {"type": "NodePort"}}'  
HubbleUiNodePort\=$(kubectl get svc \-n kube\-system hubble\-ui \-o jsonpath\={.spec.ports\[0\].nodePort})  
service/hubble\-ui patched  
(⎈|kubernetes\-admin@kubernetes:N/A) root@k8s\-s:~\# echo \-e "Hubble UI URL = http://$(curl -s ipinfo.io/ip):$HubbleUiNodePort"  
Hubble UI URL \= http:

-   The KubeProxyReplacement configuration enables Cilium to handle functions typically managed by kube-proxy, optimizing network performance.
-   eBPF masquerade provides more efficient packet handling compared to traditional iptables masquerade.
-   The Hubble UI provides valuable visibility into network flows and security policies.
-   Command aliases significantly streamline the management of Cilium across multiple nodes.

![[Raw/Media/Resources/98737c514de6ab655c79a0dc48f85bca_MD5.png]]

Hubble Client serves as Cilium’s dedicated CLI tool for real-time monitoring and analysis of Kubernetes network flows. It provides deep visibility into network communications within your cluster.

HUBBLE\_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)  
HUBBLE\_ARCH=amd64  
if \[ "$(uname -m)" = "aarch64" \]; then HUBBLE\_ARCH=arm64; fi  
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE\_VERSION/hubble-linux-${HUBBLE\_ARCH}.tar.gz{,.sha256sum}  
sha256sum --check hubble-linux-${HUBBLE\_ARCH}.tar.gz.sha256sum  
sudo tar xzvfC hubble-linux-${HUBBLE\_ARCH}.tar.gz /usr/local/bin  
rm hubble-linux-${HUBBLE\_ARCH}.tar.gz{,.sha256sum}

cilium hubble port-forward &

This command establishes port forwarding in the background, enabling local access to Hubble services. The ampersand (&) ensures the process runs in the background, allowing you to continue using your terminal.

hubble status

This command confirms the operational status of the Hubble API and its connectivity.

![[Raw/Media/Resources/f3c9c89e70cc033ddee972b876183c5c_MD5.png]]

The output reveals active communication between various components of the Kubernetes cluster. At 16:29:18, we can observe the host (192.168.10.102) maintaining regular network time synchronization through UDP connections to multiple NTP servers, including connections to 193.123.243.2:123 and 39.118.108.191:123.

There’s consistent communication with the kube-apiserver (192.168.10.10:6443), showing active cluster management operations. The traffic patterns indicate regular API server health checks and control plane operations, with TCP connections showing both ACK and PSH flags, suggesting active data transfer.

The logs also show comprehensive health checking mechanisms in action. For example, at 16:29:19, there are multiple health check communications involving port 4240, with both remote nodes and the local system participating in the health monitoring infrastructure.

The Hubble UI and Relay components demonstrate active service mesh monitoring:

-   The Hubble UI pod (hubble-ui-59bb4cb67b-dsqbv) communicates with the Hubble Relay (hubble-relay-88f7f89d4-sfnv6)
-   The relay pod maintains connections with multiple cluster nodes for data collection
-   All these communications show proper forwarding states (FORWARDED)
-   CoreDNS activity is evident through multiple traced TCP connections involving CoreDNS pods (coredns-55cb58b774-qqpkh and coredns-55cb58b774-tlcnd), indicating active DNS resolution within the cluster.
-   There are numerous pre-translation traces (pre-xlate-rev) involving localhost (127.0.0.1), particularly in communication with system services, indicating active network address translation operations.
-   All observed traffic shows appropriate FORWARDED or TRACED states, suggesting proper policy enforcement and no dropped or rejected packets in this sample, indicating well-configured network policies.

![[Raw/Media/Resources/0a89d960677b7f7ecd61d2a35f9d01c0_MD5.png]]

## Node-to-Node Pod Communication Testing Guide

First, we create a dedicated test environment and configure our context.

kubectl create ns test  
kubectl config set-context 

We deploy three distinct pods across different nodes:

1.  netpod: A network testing pod using nicolaka/netshoot image on k8s-s node
2.  webpod1: A web service pod using traefik/whoami on k8s-w1 node
3.  webpod2: A second web service pod using traefik/whoami on k8s-w2 node

c0 status   
c1 status   
c2 status 

This shows how Cilium has allocated IP addresses and network resources across nodes.

kubectl get ciliumendpoints

Displays the Cilium endpoint information for each pod, confirming proper network registration.

To streamline testing, we set up environment variables and aliases:

\# Pod IP variables  
NETPODIP=$(kubectl get pods netpod -o jsonpath='{.status.podIP}')  
WEBPOD1IP=$(kubectl get pods webpod1 -o jsonpath='{.status.podIP}')  
WEBPOD2IP=$(kubectl get pods webpod2 -o jsonpath='{.status.podIP}')

\# Command aliases  
alias p0="kubectl exec -it netpod -- "  
alias p1="kubectl exec -it webpod1 -- "  
alias p2="kubectl exec -it webpod2 -- "

p0 ip \-c \-4 addr

Shows the IPv4 address configuration within netpod.

p0 route -n

Displays the network routing table, showing how traffic is directed between pods.

p0 ping -c 1 $WEBPOD1IP && p0 ping -c 1 $WEBPOD2IP

Verifies basic network connectivity between pods using ICMP.

p0 curl -s $WEBPOD1IP && p0 curl -s $WEBPOD2IP

Tests HTTP connectivity to the web services.

p0 curl -s $WEBPOD1IP:8080 ; p0 curl -s $WEBPOD2IP:8080

Verifies service accessibility on specific ports.

p0 ping -c 1 8.8.8.8 && p0 curl -s wttr.in/seoul

Validates external network access and DNS resolution.

Each test is visualized in the Hubble UI, providing real-time network flow visibility and helping to verify:

-   Pod-to-pod communication paths
-   Network policy enforcement
-   Service discovery functionality
-   External network access
-   Protocol-specific behavior (TCP/UDP/ICMP)

![[Raw/Media/Resources/2a58dfa1119e641097dacf29a24e2332_MD5.png]]

![[Raw/Media/Resources/4eaad70ccfe555c9cf9c20ed95ac49db_MD5.png]]

![[Raw/Media/Resources/cb71249626ef525740dfa8b3aa277fa6_MD5.png]]

![[Raw/Media/Resources/23ad4f85d1ed5032ed72371567e80d69_MD5.png]]

![[Raw/Media/Resources/f9f894e06ced005ab6f4ce9cd0abadaa_MD5.png]]

![[Raw/Media/Resources/b67898af85d890882825c5a20c86f218_MD5.png]]

## Service Communication Testing and Analysis

In our Kubernetes environment, we begin by creating a Service resource to manage traffic distribution to our web pods. The Service is configured as a ClusterIP type, targeting pods labeled with ‘app: webpod’ and exposing port 80. This setup creates an abstraction layer for accessing our web applications.

apiVersion: v1  
kind: Service  
metadata:  
  name: svc  
  namespace: test  
spec:  
  ports:  
    \- name: svc-webport  
      port: 80  
      targetPort: 80  
  selector:  
    app: webpod  
  type: ClusterIP

Upon verifying the Service creation, we observe an interesting architectural shift in how network rules are handled. Traditional Kubernetes implementations would create KUBE-SVC rules in iptables, but our investigation reveals that these rules are notably absent. Instead, we find Cilium-specific rules in iptables, demonstrating Cilium’s complete handling of service networking.

iptables-save | grep KUBE-SVC

iptables-save | grep CILIUM

![[Raw/Media/Resources/bf3aedcaebffdb4c76410950dbd5a265_MD5.png]]

To facilitate our testing, we capture the Service IP address as an environment variable. This allows us to conduct systematic communication tests. When we initiate traffic from our netpod to the Service’s ClusterIP, we observe fascinating behavior in the network translation process.

SVCIP\=$(kubectl get svc svc -o jsonpath='{.spec.clusterIP}')

The continuous traffic generation test, implemented through a loop sending requests to the Service IP, reveals Cilium’s efficient load balancing capabilities. What’s particularly interesting is the network packet analysis through tcpdump within the pod. The captured traffic shows direct communication with the backend pod IPs rather than the Service ClusterIP, indicating that Cilium performs network address translation at an earlier stage in the networking stack.

kubectl exec netpod -- curl -s $SVCIP

This observation is further confirmed through ngrep analysis on port 80, where we can see that the destination IP addresses in the packets are already translated to the actual pod IPs before reaching the pod’s network interface. This demonstrates Cilium’s sophisticated approach to service networking, performing address translation more efficiently than traditional iptables-based solutions.

while true; do kubectl exec netpod -- curl -s $SVCIP | grep Hostname;echo "-----";sleep 1;done

kubectl exec netpod -- tcpdump -enni any -q

![[Raw/Media/Resources/52ee5cd46bf87830952db5c255bef179_MD5.png]]

kubectl exec netpod -- sh -c "ngrep -tW byline -d eth0 '' 'tcp port 80'"

The mechanism behind this efficient networking is revealed in the Cilium pod’s initialization process. Examining the pod specification, we find that Cilium uses a mount-cgroup init container that employs the nsenter command to access and configure namespace settings. This initialization step is crucial as it…

-   Sets up proper cgroup configurations
-   Establishes mount namespaces
-   Optimizes pod-to-pod communication paths
-   Enables Cilium’s efficient traffic management capabilities

![[Raw/Media/Resources/389e5c71dffa5416b057034f2539e15e_MD5.png]]

kubectl describe pod \-n kube\-system cilium\-s4hq5

This architecture represents a significant advancement over traditional Kubernetes networking, offering…

-   Reduced network processing overhead
-   More efficient service discovery
-   Optimized packet routing
-   Better performance through early packet translation
-   Reduced complexity in the networking stack

The absence of traditional KUBE-SVC iptables rules and the direct pod-to-pod communication paths demonstrate how Cilium leverages eBPF to provide a more efficient networking solution, fundamentally changing how service networking operates in the Kubernetes cluster.

![[Raw/Media/Resources/866bd58f694a0483da28c44df4c3f963_MD5.png]]

## Monitoring Infrastructure for Cilium

Cilium provides robust monitoring capabilities through the integration of Prometheus for metrics collection and Grafana for visualization. This monitoring setup allows administrators to gain deep insights into network performance, policy enforcement, and overall cluster health.

The monitoring infrastructure can be quickly deployed using Cilium’s prepared monitoring example configuration. The deployment process involves applying a comprehensive YAML configuration that sets up both Prometheus and Grafana in the cilium-monitoring namespace.

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.16.3/examples/kubernetes/addons/prometheus/monitoring-example.yaml

To make these monitoring services accessible from outside the cluster, we modify their service configurations to use NodePort:

kubectl patch svc grafana -n cilium-monitoring -p '{"spec": {"type": "NodePort"}}'  
kubectl patch svc prometheus -n cilium-monitoring -p '{"spec": {"type": "NodePort"}}'

After deployment, both Grafana and Prometheus web interfaces become accessible through their respective NodePorts. The access URLs can be constructed using the cluster’s external IP address and the assigned NodePort numbers:

For Grafana:

GPT=$(kubectl get svc -n cilium-monitoring grafana -o jsonpath={.spec.ports\[0\].nodePort})  
echo -e "Grafana URL = http://$(curl -s ipinfo.io/ip):$GPT"

For Prometheus:

PPT=$(kubectl get svc -n cilium-monitoring prometheus -o jsonpath={.spec.ports\[0\].nodePort})  
echo -e "Prometheus URL = http://$(curl -s ipinfo.io/ip):$PPT"

The Prometheus web interface provides direct access to metric data, allowing administrators to query and analyze various performance indicators. The included monitoring example Helm chart comes pre-configured with Cilium-specific dashboards in Grafana, offering immediate visibility into Cilium’s operation.

These dashboards present comprehensive metrics about Cilium CNI’s performance and operational status. The integration with Prometheus enables Grafana to display the same metrics available in the Hubble UI, providing administrators with multiple ways to view and analyze network performance data.

The combined use of Prometheus and Grafana creates a powerful monitoring solution that helps administrators…

-   Track network performance metrics
-   Monitor policy enforcement
-   Analyze traffic patterns
-   Identify potential issues
-   Maintain optimal cluster operation

This monitoring setup ensures that administrators have comprehensive visibility into their Cilium-managed network infrastructure, enabling proactive management and quick problem resolution.

![[Raw/Media/Resources/b62a0500b08d72e78363eb77e890738d_MD5.png]]

![[Raw/Media/Resources/0309b15bba775052985c3b56884f7831_MD5.png]]

![[Raw/Media/Resources/3c7438f39a19895b336941736a1b17aa_MD5.png]]

## Understanding Cilium Network Policies

Cilium implements a sophisticated multi-layer network policy framework that operates at three distinct levels:

Layer 3 (Identity-Based) Control: Cilium leverages Kubernetes pod labels to establish identity-based access control between endpoints. This fundamental layer allows administrators to create logical groupings of pods and control their interactions. For example, pods labeled with ‘role=frontend’ can be specifically permitted to communicate with pods labeled ‘role=backend’, creating clear communication boundaries based on pod identity.

Layer 4 (Port-Based) Control: At the transport layer, Cilium provides granular control over network communications based on port numbers and protocols. Administrators can define precise rules about which ports can accept incoming connections and which ports can be used for outgoing connections. For instance, frontend pods might be restricted to only establish outbound connections on port 443 (HTTPS), while backend pods might only accept incoming connections on specific service ports.

![[Raw/Media/Resources/295372b6e5cb4c372580059fb30a9f8b_MD5.png]]

To illustrate these concepts in practice, we implement a Star Wars-themed demonstration that showcases Cilium’s network policy enforcement.

![[Raw/Media/Resources/3a828cc7984ff6888eaa65765350459a_MD5.png]]

  
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/1.16.3/examples/minikube/http-sw-app.yaml

We deploy a sample application that includes several components:

-   A Death Star service (representing the Empire’s battle station)
-   TIE Fighter pods (representing Empire spacecraft)
-   X-wing pods (representing Rebellion spacecraft)

Initially, we verify that both spacecraft types can access the Death Star by sending POST requests to the landing endpoint.

  
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing

  
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing

Without any network policies in place, both the X-wing and TIE Fighter successfully complete their landing requests, demonstrating unrestricted access.

![[Raw/Media/Resources/f11bd9008d3482014defc4d481ea34de_MD5.png]]

![[Raw/Media/Resources/e8e202ee441576c5859f9b3c6fb92483_MD5.png]]

![[Raw/Media/Resources/58193a0b35b4b6ee2e154e2f3bc9d10b_MD5.png]]

We then deploy a CiliumNetworkPolicy that implements specific access controls:

-   The policy targets the Death Star using labels org: empire and class: deathstar
-   It only allows incoming connections from endpoints labeled with org: empire
-   The policy specifically permits TCP traffic on port 80

The policy definition creates a clear security boundary:

-   TIE Fighters, being part of the Empire (labeled with org: empire), maintain their ability to land
-   X-wings, part of the Rebellion, are now blocked from accessing the Death Star

The Hubble UI provides immediate visual confirmation of the policy enforcement:

-   Successful connections from TIE Fighters appear as allowed traffic
-   Blocked connection attempts from X-wings are clearly visible
-   The policy enforcement happens in real-time, with immediate effect

This demonstration effectively shows how Cilium can implement sophisticated network policies that combine identity-based access control with traditional port-based restrictions, all while providing real-time visibility into policy enforcement through the Hubble interface.

![[Raw/Media/Resources/54ca4398cbf292468db40bce5d449f4b_MD5.png]]

![[Raw/Media/Resources/6a76123929bec9efba23097859f0ce2a_MD5.png]]

![[Raw/Media/Resources/e51cec174a9a984dfa52b5880f021b05_MD5.png]]

## L2 Announcements / L2 Aware LB (beta)

L2 Announcements is a sophisticated feature designed for making services accessible in local area networks, particularly beneficial for on-premises deployments without BGP-based routing. This feature operates at Layer 2 of the network stack, handling ARP queries for ExternalIP and LoadBalancer IP addresses.

![[Raw/Media/Resources/2f830e65605d8559473eb60f3b1c47f2_MD5.png]]

[https://yeoli-tech.tistory.com/51#%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC](https://yeoli-tech.tistory.com/51#%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC)

-   Virtual IP Management: Manages IPs across multiple nodes without physical network device installation
-   Coordinated Response: Ensures only one node responds to ARP queries at a time
-   Load Balancing: Performs north-south load balancing through service load balancing functionality
-   Port Flexibility: Allows multiple services to use the same port numbers through unique IP assignments
-   High Availability: Enables seamless VIP migration between nodes during failures

**Enabling L2 Announcements**

helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values \\  
\--set l2announcements.enabled=true --set externalIPs.enabled=true \\  
\--set l2announcements.leaseDuration

We implement a CiliumL2AnnouncementPolicy that:

-   Selects services based on labels
-   Specifies eligible nodes for announcement
-   Defines network interfaces for announcements
-   Enables support for external and load balancer IPs

![[Raw/Media/Resources/764111b2437bae12c1166df1483d2600_MD5.png]]

cat <<EOF | kubectl apply \-f \-  
apiVersion: v1  
kind: Pod  
metadata:  
  name: webpod1  
  labels:  
    app: webpod  
spec:  
  nodeName: k8s-w1  
  containers:  
  \- name: container  
    image: traefik/whoami  
  terminationGracePeriodSeconds: 0  
\---  
apiVersion: v1  
kind: Pod  
metadata:  
  name: webpod2  
  labels:  
    app: webpod  
spec:  
  nodeName: k8s-w2  
  containers:  
  \- name: container  
    image: traefik/whoami  
  terminationGracePeriodSeconds: 0  
\---  
apiVersion: v1  
kind: Service  
metadata:  
  name: svc1  
spec:  
  ports:  
    \- name: svc1-webport  
      port: 80  
      targetPort: 80  
  selector:  
    app: webpod  
  type: LoadBalancer    
\---  
apiVersion: v1  
kind: Service  
metadata:  
  name: svc2  
spec:  
  ports:  
    \- name: svc2-webport  
      port: 80  
      targetPort: 80  
  selector:  
    app: webpod  
  type: LoadBalancer  
\---  
apiVersion: v1  
kind: Service  
metadata:  
  name: svc3  
spec:  
  ports:  
    \- name: svc3-webport  
      port: 80  
      targetPort: 80  
  selector:  
    app: webpod  
  type: LoadBalancer  
EOF

IP Pool Configuration: We establish a CiliumLoadBalancerIPPool with specific CIDR ranges for IP allocation.

![[Raw/Media/Resources/1d9b2dbaed38f27d0cb11dfbae141d29_MD5.png]]

XDP integration in Cilium requires compatible Network Interface Controllers or Elastic Network Interfaces. For AWS environments, this means understanding the various networking options:

Network Interface Types in AWS is the following: ENI (Basic), ENA (Enhanced) and EFA (Elastic Fabric Adapter).

-   Performance hierarchy: ENI < ENA < EFA
-   AWS XDP Support Requirements: ENA driver version 2.2.0 or later (introduced January 2020), AWS Nitro System support, Compatible instance types.

ifconfig  
ethtool -i ens5

![[Raw/Media/Resources/a02ed8c24e468e2fad1487b01622607e_MD5.png]]

![[Raw/Media/Resources/da271af383015eb91df6c551c3ba0da6_MD5.png]]

  
apt upgrade  
apt install -y -q awscli 

  
AMI\_ID=$(curl 169.254.169.254/latest/meta-data/ami-id)  
aws ec2 describe-images --image-id $AMI\_ID --query "Images\[\].EnaSupport"

  
ethtool -i ens5

Please note that the Best Practices is the following.

-   Use instances built on AWS Nitro network system
-   Ensure ENA driver version compatibility
-   Verify hardware support through ethtool
-   Consider instance type selection based on networking requirements

The practical implementation demonstrates how Cilium leverages these features to provide the following.

-   Enhanced network performance
-   Improved packet processing
-   Better load balancing capabilities
-   Increased network security