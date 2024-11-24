---
title: Kubernetes network learning with Cilium and eBPF
source: https://addozhang.medium.com/kubernetes-network-learning-with-cilium-and-ebpf-aafbf3163840
clipped: 2024-02-05
published: 
category: network
tags:
  - k8s
read: false
---


![[Raw/Media/Resources/2a300c41ec3c425bc7c6b21913b2d6b8_MD5.png]]

This is the fifth installment in the series on Kubernetes networking learning, and it is planned to be the last one as previously outlined.

-   [Deep Dive into Kubernetes Network Model and Communication](https://medium.com/@addozhang/deep-dive-into-kubernetes-network-model-and-communication-57a2bffc852e)
-   [Understanding the Container Network Interface (CNI)](https://addozhang.medium.com/introduction-to-container-network-interface-cni-25309a64b23e)
-   [Source code analysis: how kubelet and container runtime work with CNI](https://medium.com/@addozhang/source-code-analysis-understanding-cnis-usage-from-kubelet-container-runtime-24d72f29466b)
-   [Learning Kubernetes VXLAN network from Flannel](https://addozhang.medium.com/learning-kubernetes-vxlan-network-with-flannel-2d6a58c95300)
-   [Kubernetes network learning with Cilium and eBPF](https://addozhang.medium.com/kubernetes-network-learning-with-cilium-and-ebpf-aafbf3163840)

Last year, I posted an article titled [“Enhancing Kubernetes Network Security with Cilium”](https://atbug.com/enhance-kubernetes-network-security-with-cilium/) having had some exposure to Cilium, utilizing Cilium’s network policies to restrict communication between pods at the network level. However, at that time, I did not delve into its implementation principles, nor did I have a deep understanding of Kubernetes networking and CNI. This time, we explore Cilium’s network through a practical environment.

The Cilium version used in this article is v1.12.3, the operating system is Ubuntu 20.04, and the kernel version is 5.4.0–91-generic.

> [*Cilium*](https://cilium.io/) *is an open-source software designed to provide, protect, and observe network connectivity between container workloads (cloud-native) powered by the revolutionary kernel technology* [*eBPF*](https://ebpf.io/)*.*

![[Raw/Media/Resources/811c119d47d8f9642b852a1ff492156e_MD5.png]]

> *The Linux kernel has always been an ideal place for implementing monitoring/observability, networking, and security features. However, in many cases, this is not an easy task as it requires modifying kernel source code or loading kernel modules, ultimately adding new abstractions on top of existing ones. eBPF is a revolutionary technology that enables running sandboxed programs within the kernel without the need for modifying kernel source code or loading kernel modules.*
> 
> *Making the Linux kernel programmable allows the creation of more intelligent, feature-rich infrastructure software based on existing (rather than adding new) abstraction layers, without increasing system complexity, sacrificing execution efficiency, or compromising security.*

Linux’s kernel provides a set of BPF hooks on the network stack, which can trigger the execution of BPF programs. Cilium datapath utilizes these hooks to load BPF programs, creating a more advanced network structure.

Upon reading the [Cilium Reference Documentation on eBPF Datapath](https://docs.cilium.io/en/stable/concepts/ebpf/intro/), it’s clear that Cilium utilizes the following hooks:

-   **XDP**: This is a hook in the network driver that can trigger BPF programs when network packets are received, and is the earliest point of interception. Since no other operations have been performed at this point, such as writing the network packet into memory, it is highly suitable for running filters to discard malicious or unintended traffic, along with other common DDOS protection mechanisms.
-   **Traffic Control Ingress/Egress**: BPF programs attached to the traffic control (abbreviated as tc) ingress hooks can also be attached to network interfaces. This hook executes before the network stack at Layer 3 (L3) and can access most of the metadata of network packets. It’s suitable for handling operations on the local node, such as applying L3/L4 endpoint policies\[¹\], forwarding traffic to endpoints. CNI often uses virtual ethernet interfaces (`veth`) to connect containers to the host's network namespace. By using a tc ingress hook attached to the host-side `veth`, all traffic leaving the container can be monitored and policies can be enforced. Meanwhile, by attaching another BPF program to the tc egress hook, Cilium can monitor all traffic entering and exiting the node and enforce policies.
-   **Socket operations**: The socket operations hook is attached to a specific cgroup and runs on TCP events. Cilium attaches a BPF socket operations program to the root cgroup and uses it to monitor TCP state transitions, especially the ESTABLISHED state transitions. When the socket state becomes ESTABLISHED, if the TCP socket’s peer is also on the current node (or possibly a local proxy), the Socket send/recv programs will be attached.
-   **Socket send/recv**: This hook runs on every send operation performed by a TCP socket. At this point, the hook can inspect messages and either discard them, send them to the TCP layer, or redirect them to another socket. Cilium uses it to accelerate data path redirection.

Since these will be utilized later on, we’ve placed emphasis on explaining these hooks.

In previous articles, I used k3s and manually installed the CNI plugins to set up the experimental environment. This time, we are using [k8e](https://getk8e.com/) directly, as k8e employs Cilium as the default CNI implementation.

I am still setting up a dual-node cluster (`ubuntu-dev2: 192.168.1.12`, `ubuntu-dev3: 192.168.1.13`) on my homelab.

Master node:

curl -sfL https://getk8e.com/install.sh | API\_SERVER\_IP=192.168.1.12 K8E\_TOKEN=ilovek8e INSTALL\_K8E\_EXEC="server --cluster-init --write-kubeconfig-mode 644 --write-kubeconfig ~/.kube/config" sh -

Worker node:

curl -sfL https://getk8e.com/install.sh | K8E\_TOKEN=ilovek8e K8E\_URL=https://192.168.1.12:6443 sh -

Deploy the sample application and schedule it on different nodes:

```yaml
NODE1=ubuntu-dev2  
NODE2=ubuntu-dev3  
kubectl apply \-n default \-f \- <<EOF  
apiVersion: v1  
kind: Pod  
metadata:  
  labels:  
    app: curl  
  name: curl  
spec:  
  containers:  
  \- image: curlimages/curl  
    name: curl  
    command: \["sleep", "365d"\]  
  nodeName: $NODE1  
\---  
apiVersion: v1  
kind: Pod  
metadata:  
  labels:  
    app: httpbin  
  name: httpbin  
spec:  
  containers:  
  \- image: kennethreitz/httpbin  
    name: httpbin  
  nodeName: $NODE2  
EOF
```

For ease of use, set the sample application, cilium pod and other information as environment variables:

```
NODE1\=ubuntu-dev2  
NODE2\=ubuntu-dev3

cilium1=$(kubectl get po -n kube-system -l k8s-app=cilium --field-selector spec.nodeName=$NODE1 -o jsonpath='{.items\[0\].metadata.name}')  
cilium2=$(kubectl get po -n kube-system -l k8s-app=cilium --field-selector spec.nodeName=$NODE2 -o jsonpath='{.items\[0\].metadata.name}')
```

Following the usual routine, start tracing the network packets from the request initiator. This time, use `Service` for access: `curl [http://10.42.0.51:80/get](http://10.42.0.51/get.)`[.](http://10.42.0.51/get.)

```
kubectl get po httpbin \-n default \-o wide  
NAME      READY   STATUS    RESTARTS   AGE   IP           NODE          NOMINATED NODE   READINESS GATES  
httpbin   1/1     Running   0          3m   10.42.0.51   ubuntu\-dev3   <none\>           <none\>
```

Check the routing table of pod `curl`:

```
kubectl exec curl \-n default   
10.42.0.51 via 10.42.1.247 dev eth0  src 10.42.1.80
```

It is known that the network packet is sent to the Ethernet interface `eth0`, and then its MAC address `ae:36:76:3e:c3:03` is found using arp:

```
kubectl exec curl \-n default   
? (10.42.1.247) at ae:36:76:3e:c3:03 \[ether\]  on eth0
```

Check the information of interface `eth0`:

```
kubectl exec curl -n default -- ip link show eth0  
42: eth0@if43: <BROADCAST,MULTICAST,UP,LOWER\_UP,M-DOWN> mtu 1500 qdisc noqueue state UP qlen 1000  
    link/ether f6:00:50:f9:92:a1 brd ff:ff:ff:ff:ff:ff
```

It is found that its MAC address is not `ae:36:76:3e:c3:03`. From the `@if43` in the name, we can know that the index of its `veth` pair is `43`, and then log in to the node`NODE1`\*\* Query the information of the index interface:

```
ip link | grep -A1 ^43  
43: lxc48c4aa0637ce@if42: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether ae:36:76:3e:c3:03 brd ff:ff:ff:ff:ff:ff link\-netns cni-407cd7d8-7c02-cfa7-bf93-22946f923ffd
```

We see that the MAC of this interface `lxc48c4aa0637ce` is exactly `ae:36:76:3e:c3:03`.

According to [past experience](https://atbug.com/deep-dive-k8s-network-mode-and-communication/), this virtual Ethernet interface `lxc48c4aa0637ce` is a **virtual Ethernet port**, located in the host's root network namespace. On one hand, it is connected to the container's Ethernet interface `eth0` via a tunnel, and packets sent to either end will reach the other end directly; on the other hand, it should be connected to a bridge in the host namespace, but the name of the bridge is not found from the above results.

Check with `ip link`:

```
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
2: eth0: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc fq\_codel state UP mode DEFAULT group default qlen 1000  
    link/ether fa:cb:49:4a:28:21 brd ff:ff:ff:ff:ff:ff  
3: cilium\_net@cilium\_host: <BROADCAST,MULTICAST,NOARP,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether 36:d5:5a:2a:ce:80 brd ff:ff:ff:ff:ff:ff  
4: cilium\_host@cilium\_net: <BROADCAST,MULTICAST,NOARP,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether 12:82:fb:78:16:6a brd ff:ff:ff:ff:ff:ff  
5: cilium\_vxlan: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/ether fa:42:4d:22:b7:d0 brd ff:ff:ff:ff:ff:ff  
25: lxc\_health@if24: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether 3e:4f:b3:56:67:2b brd ff:ff:ff:ff:ff:ff link-netnsid 0  
33: lxc113dd6a50a7a@if32: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether 32:3a:5b:15:44:ff brd ff:ff:ff:ff:ff:ff link-netns cni-07cffbd8-83dd-dcc1-0b57-5c59c1c037e9  
43: lxc48c4aa0637ce@if42: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether ae:36:76:3e:c3:03 brd ff:ff:ff:ff:ff:ff link-netns cni-407cd7d8-7c02-cfa7-bf93-22946f923ffd
```

We see multiple Ethernet interfaces: `cilium_net`, `cilium_host`, `cilium_vxlan`, `cilium_health` and the tunnel endpoints `lxcxxxx` associated with the container network namespace.

![[Raw/Media/Resources/784f6d42546186d32f45184765a9830a_MD5.png]]

How does the network packet proceed from `lxcxxx`? Now it's the turn for eBPF to come into play.

Note that `cilium_net`, `cilium_host`, and `cilium_health` will not be discussed in the text, thus they will not be represented in the following diagram.

Enter the cilium pod on the current node, which is the variable `$cilium1` set earlier, and use the `bpftool`command to check the BPF program attached to this veth.

kubectl exec \-n kube\-system $cilium1 \-c cilium\-agent   
xdp:

tc:  
lxc48c4aa0637ce(43) clsact/ingress bpf\_lxc.o:\[from-container\] id 2901flow\_dissector:

One can also log into node `$NODE1` and use the `tc` command to inquire. Note that here we specified `ingress` in the datapath section at the beginning of the article. Since the container's `eth0` and the host network namespace's `lxc` constitute a channel, the container's egress (Egress) traffic is the `lxc` ingress `Ingress` traffic. Similarly, the container's ingress traffic is the `lxc` egress traffic.

  
tc filter show dev lxc48c4aa0637ce ingress  
filter protocol all pref 1 bpf chain 0  
filter protocol all pref 1 bpf chain 0 handle 0x1 bpf\_lxc.o:\[from\-container\] direct-action not\_in\_hw id 2901 tag d578585f7e71464b jited

Details can be viewed via program `id 2901`.

kubectl exec \-n kube\-system $cilium1 \-c cilium\-agent   
2901: sched\_cls  name handle\_xgress  tag d578585f7e71464b  gpl  
    loaded\_at 2023\-01\-09T19:29:52+0000  uid 0  
    xlated 688B  jited 589B  memlock 4096B  map\_ids 572,86  
    btf\_id 301

It can be seen that the `from-container` part of the BPF program `bpf_lxc.o` is loaded here. Go to the `__section("from-container")` part of Cilium's source code [bpf\_lxc.c](https://github.com/cilium/cilium/blob/1c466d26ff0edfb5021d024f755d4d00bc744792/bpf/bpf_lxc.c#L1320), program name`handle_xgress`:

handle\_xgress #1  
  validate\_ethertype(ctx, &proto)  
  tail\_handle\_ipv4 #2  
    handle\_ipv4\_from\_lxc #3  
      lookup\_ip4\_remote\_endpoint => ipcache\_lookup4 #4  
      policy\_can\_access #5  
      if TUNNEL\_MODE #6  
        encap\_and\_redirect\_lxc  
          ctx\_redirect(ctx, ENCAP\_IFINDEX, 0)  
      if ENABLE\_ROUTING  
        ipv4\_l3  
      return CTX\_ACT\_OK;

(1): The header information of the network packet is sent to `handle_xgress`, and then its L3 protocol is checked.

(2): All IPv4 network packets are handed over to `tail_handle_ipv4` for processing.

(3): The core logic is in `handle_ipv4_from_lxc`. How `tail_handle_ipv4` jumps to `handle_ipv4_from_lxc` is facilitated by [Tails Call](https://docs.cilium.io/en/stable/bpf/#tail-calls). Tails Call allows us to configure a specified program to execute upon the completion of a certain BPF program and under certain conditions, without needing to return to the original program. For further details, those interested can refer to the [official documentation](https://docs.cilium.io/en/stable/bpf/#tail-calls).

(4): Next, query the target endpoint from the eBPF map\[²\] `cilium_ipcache`, and find the tunnel endpoint `192.168.1.13`, which is the IP address of the target node, with type.

kubectl exec -n kube-system $cilium1 -c cilium-agent -- cilium map get cilium\_ipcache | grep 10.42.0.51  
10.42.0.51/32     identity=15773 encryptkey=0 tunnelendpoint=192.168.1.13   sync

(5): `policy_can_access` here performs the egress policy check, which is not discussed in this article and will not be elaborated.

(6): Subsequent processing has three modes:

-   Direct Routing: Handed over to the kernel network stack for processing, or supported by underlaying SDN.
-   Tunneling: The network packet is re-encapsulated and transmitted through a tunnel, such as vxlan.
-   Using Proxy: If `policy_can_access` returns a proxy port in #5 (if the network policy acts on L7, the policy automatically specifies the proxy port), a proxy (Node level) is used for routing. For this part, refer to another article [How Cilium Processes L7 Traffic](https://atbug.com/deep-dive-into-cilium-l7-packet-processing/), which provides a deep dive into the L7 traffic policy execution using a proxy.

Here we are also using tunnel mode. The network packet is handed over to `encap_and_redirect_lxc` for processing, using the tunnel endpoint as the tunnel peer. Finally, it's forwarded to `ENCAP_IFINDEX` (this value is the index of the interface, obtained when cilium-agent starts), which is the Ethernet interface `cilium_vxlan`.

Let’s first take a look at the BPF program on this interface.

kubectl exec \-n kube\-system $cilium1 \-c cilium\-agent   
xdp:

tc:  
cilium\_vxlan(5) clsact/ingress bpf\_overlay.o:\[from-overlay\] id 2699  
cilium\_vxlan(5) clsact/egress bpf\_overlay.o:\[to-overlay\] id 2707flow\_dissector:

The egress traffic of the container is also egress for `cilium_vxlan`, so the program here is `to-overlay`.

The program is located in `[bpf_overlay.c](https://github.com/cilium/cilium/blob/1c466d26ff0edfb5021d024f755d4d00bc744792/bpf/bpf_overlay.c#L528)`. The processing of this program is straightforward. If it's IPv6 protocol, the packet will be encapsulated with an IPv6 address. As it's IPv4 here, it directly returns `CTX_ACT_OK`. The network packet is handed over to the kernel network stack, entering the `eth0` interface.

Let’s first take a look at the BPF program.

kubectl exec \-n kube\-system $cilium1 \-c cilium\-agent   
xdp:

tc:  
eth0(2) clsact/ingress bpf\_netdev\_eth0.o:\[from-netdev\] id 2823  
eth0(2) clsact/egress bpf\_netdev\_eth0.o:\[to-netdev\] id 2832flow\_dissector:

The egress program `to-netdev` is located in `[bpf_host.c](https://github.com/cilium/cilium/blob/1c466d26ff0edfb5021d024f755d4d00bc744792/bpf/bpf_host.c#L1081)`. In fact, it doesn't do any significant processing, but just returns `CTX_ACT_OK` to allow the kernel network stack to continue processing: sending the network packet through the vxlan tunnel to the peer, which is the node `192.168.1.13`. The data transmission in between actually still utilizes the underlaying network, going from the host's `eth0` interface through the underlaying network to the target host's `eth0` interface.

The vxlan network packet reaches the `eth0` interface of the node, which also triggers the BPF program.

kubectl exec \-n kube\-system $cilium2 \-c cilium\-agent   
xdp:

tc:  
eth0(2) clsact/ingress bpf\_netdev\_eth0.o:\[from-netdev\] id 4556  
eth0(2) clsact/egress bpf\_netdev\_eth0.o:\[to-netdev\] id 4565flow\_dissector:

This time the trigger is `from-netdev`, located in [bpf\_host.c](https://github.com/cilium/cilium/blob/1c466d26ff0edfb5021d024f755d4d00bc744792/bpf/bpf_host.c#L1040).

from\_netdev  
  if vlan  
    allow\_vlan  
    return CTX\_ACT\_OK

For the vxlan tunnel mode, the logic here is quite straightforward. Once it determines the network packet is vxlan and confirms that vlan is allowed, it directly returns `CTX_ACT_OK` to hand the processing over to the kernel network stack.

The network packet, through the kernel network stack, has arrived at the interface `cilium_vxlan`.

kubectl exec \-n kube\-system $cilium2 \-c cilium\-agent   
xdp:

tc:  
cilium\_vxlan(5) clsact/ingress bpf\_overlay.o:\[from-overlay\] id 4468  
cilium\_vxlan(5) clsact/egress bpf\_overlay.o:\[to-overlay\] id 4476flow\_dissector:

The program is located in `[bpf_overlay.c](https://github.com/cilium/cilium/blob/1c466d26ff0edfb5021d024f755d4d00bc744792/bpf/bpf_overlay.c#L430)`.

from\_overlay  
  validate\_ethertype  
    tail\_handle\_ipv4  
      handle\_ipv4  
        lookup\_ip4\_endpoint 1  
          map\_lookup\_elem  
        ipv4\_local\_delivery 2  
          tail\_call\_dynamic 3

(1): `lookup_ip4_endpoint` will check in the eBPF map `cilium_lxc` to see if the destination address is in the current node (this map only saves the endpoints in the current node).

kubectl exec -n kube-system $cilium2 -c cilium-agent -- cilium map get cilium\_lxc | grep 10.42.0.51  
10.42.0.51:0    id=2826  flags=0x0000 ifindex=29  mac=96:86:44:A6:37:EC nodemac=D2:AD:65:4D:D0:7B   sync

Here it finds the information of the target endpoint: id, Ethernet port index, and MAC address. On the NODE2 node, upon checking the interface information, it’s found that this port is the virtual Ethernet device `lxc65015af813d1`, which happens to be the counterpart to the `eth0` interface of the pod `httpbin`.

ip link | grep -B1 -i d2:ad  
29: lxc65015af813d1@if28: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether d2:ad:65:4d:d0:7b brd ff:ff:ff:ff:ff:ff link\-netns cni-395674eb-172b-2234\-a9ad-1db78b2a5beb

kubectl exec -n default httpbin -- ip link  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
28: eth0@if29: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether 96:86:44:a6:37:ec brd ff:ff:ff:ff:ff:ff link-netnsid

(2): The logic of `ipv4_local_delivery` is located in `[l3.h](https://github.com/cilium/cilium/blob/1c466d26ff0edfb5021d024f755d4d00bc744792/bpf/lib/l3.h#L114)`, where it will tail-call the BPF program located by the endpoint's LXC ID (`29`).

Executing the following command will not find the expected egress `to-container` (compared to `from-container`).

kubectl exec \-n kube\-system $cilium2 \-c cilium\-agent   
lxc65015af813d1(29) clsact/ingress bpf\_lxc.o:\[from\-container\] id 4670

The BPF programs used earlier were attached to the interfaces, but here there is a program directly attached to vxlan that’s being tail-called. `to-container` can be found in `[bpf-lxc.c](https://github.com/cilium/cilium/blob/1c466d26ff0edfb5021d024f755d4d00bc744792/bpf/bpf_lxc.c#L2131)`.

handle\_to\_container  
  tail\_ipv4\_to\_endpoint  
    ipv4\_policy   
      policy\_can\_access\_ingress  
    redirect\_ep  
      ctx\_redirect

(1): `ipv4_policy` will execute the configured policy.

(2): If the policy passes, `redirect_ep` will be called to send the network packet to the virtual Ethernet interface `lxc65015af813d1`. Upon entering the veth (Virtual Ethernet), it will directly reach the connected container's `eth0` interface.

The network packet reaches pod2, attaching a completed diagram here.

![[Raw/Media/Resources/911f7055e446308884051365fd495cde_MD5.png]]

In my opinion, the content covered in this article is just the tip of the iceberg when it comes to Cilium. For someone like me, who lacks knowledge in kernel and C language, diving into it is extremely challenging. There’s so much more to Cilium that I haven’t delved into deeply yet. I can’t help but marvel at how complex Cilium is. From what I understand so far, Cilium maintains its own set of data in BPF maps, including endpoints, nodes, policies, routes, connection statuses, and a lot more, all stored within the kernel. Moreover, the development and maintenance costs of BPF programs increase with their complexity. It’s hard to imagine how complicated it would be to develop L7 functionalities using BPF programs. This is probably why proxies are used to handle L7 scenarios.

Let me share some experiences from my journey learning about Cilium.

First, when it comes to reading BPF programs, the `bpf` code in the project is static, containing many configuration-related `if else` statements. At runtime, it gets compiled based on the configuration. Under such circumstances, one can go into the Cilium pod and find the source files with applied configurations in the `/run/cilium/state/templates` directory; the amount of code is much less there. The current configuration in use can be found in `/run/cilium/state/globals/node_config`, which can be used in tandem with the code to understand it better.

**Footnotes**

\[¹\]: Cilium makes containers available on the network by assigning them IP addresses. Multiple containers can share the same IP address, just like how multiple containers within a Kubernetes Pod can share the same network namespace and use the same IP address. For these containers that share an address, Cilium bundles them together, designating them as an Endpoint. \[²\]: eBPF’s map can be used to store data. In Cilium, the cilium-agent monitors the api-server and writes information into the map. For instance, the `cilium_lb4_services_v2` maintains all the information related to Kubernetes `Service`.