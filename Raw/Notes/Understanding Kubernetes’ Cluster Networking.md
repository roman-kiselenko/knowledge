---
title: Understanding Kubernetes’ Cluster Networking
source: https://llorllale.github.io/posts/k8s-cluster-network/
clipped: 2023-09-04
published: 
category: network
tags:
  - k8s
  - network
read: true
---

[![cover](https://llorllale.github.io/assets/img/Kubernetes-icon-color.svg)](https://llorllale.github.io/assets/img/Kubernetes-icon-color.svg) [Kubernetes](https://kubernetes.io/) is a system for automating deployment, scaling, and management of containerized applications. [Networking is a central part of Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/networking/), and in this article we will explore how Kubernetes configures the cluster to handle [east-west traffic](https://kubernetes.io/docs/concepts/cluster-administration/networking/). We’ll reserve discussion on north-south traffic for a later article.

> This article is long and a bit heavy-handed on annotations, command-line instructions, and pointers to implementations in Kubernetes and associated components. There are dozens of footnotes. We are diving into fairly deep waters here. I tried my best to keep a coherent flow going. Feel free to drop a comment below if you notice a mistake somewhere or if you’d like to offer editorial advice.

By default, all pods in a K8s cluster can communicate with each other without [NAT](https://en.wikipedia.org/wiki/Network_address_translation) ([source](https://kubernetes.io/docs/concepts/services-networking/))[1](#fn:1), therefore each pod is assigned a cluster-wide IP address. Containers within each pod share the pod’s network namespace, allowing them to communicate with each other on `localhost` via the `loopback` interface. From the point of view of the workloads running inside the containers, this IP network looks like any other and no changes are necessary.

[![k8s-pod-container-network](https://llorllale.github.io/assets/img/k8s-networking/k8s-pod-container-network.svg)](https://llorllale.github.io/assets/img/k8s-networking/k8s-pod-container-network.svg "Conceptual view of inter-Pod and intra-Pod network communication.") *Conceptual view of inter-Pod and intra-Pod network communication.*

Recall from a previous article that as far as K8s components go, the [kubelet and the kube-proxy](https://llorllale.github.io/posts/kubernetes-in-action/#node-components) are responsible for creating pods and applying network configurations on the cluster’s nodes.

When the pod is being created or terminated, part of the `kubelet`’s job is to set up or cleanup the pod’s sandbox on the node it is running on. The `kubelet` relies on the [Container Runtime Interface](https://github.com/kubernetes/cri-api) (CRI) implementation to handle the details of creating and destroying sandboxes. The CRI is composed of several interfaces; the interesting ones for us are the [`RuntimeService`](https://github.com/kubernetes/cri-api/blob/adbbc6d75b383d6b823c24bba946029458d6681b/pkg/apis/services.go#L106-L118) interface (client-side API; integration point `kubelet`\->CRI) and the [`RuntimeServiceServer`](https://github.com/kubernetes/cri-api/blob/adbbc6d75b383d6b823c24bba946029458d6681b/pkg/apis/runtime/v1/api.pb.go#L10453-L10543) interface (server-side API; integration point `RuntimeService`\->CRI implementation). These APIs are both big and fat, but for this article we are only interested in the `*PodSandbox` set of methods (e.g. `RunPodSandbox`). Underneath the CRI’s hood, however, is the [Container Network Interface](https://github.com/containernetworking/cni) that creates and configures the pod’s [network namespace](https://en.wikipedia.org/wiki/Linux_namespaces#Network_(net))[2](#fn:11).

The `kube-proxy` configures routing rules to proxy traffic directed at [`Services`](https://kubernetes.io/docs/concepts/services-networking/service/) and performs simple load-balancing between the corresponding [`Endpoints`](https://kubernetes.io/docs/concepts/services-networking/service/#endpoints)[3](#fn:6).

Finally, a third component, [`coreDNS`](https://github.com/coredns/coredns), resolves network names by looking them up in `etcd`.

[![k8s-pod-sandbox-network](https://llorllale.github.io/assets/img/k8s-networking/k8s-cri-network.svg)](https://llorllale.github.io/assets/img/k8s-networking/k8s-cri-network.svg "Components involved in the network configuration for a pod. Blue circles are pods and orange rectangles are daemons. Note that etcd is shown here as a database service, but it is also deployed as a pod.") *Components involved in the network configuration for a pod. Blue circles are pods and orange rectangles are daemons. Note that `etcd` is shown here as a database service, but it is also deployed as a pod.*

In the next section we will understand how pod networking works by manually creating our own pods and have a client in one pod invoke an API in a different pod.

> I will be using a simple K8s cluster I set up with [`kind`](https://github.com/kubernetes-sigs/kind) in the walkthrough below. `kind` creates a docker container per K8s node. You may choose a similar sandbox, machine instances in the cloud, or any other setup that simulates at least two host machines connected to the same network. Also note that Linux hosts are used for this walkthrough.

We will manually create pods on different hosts to gain an understanding of how Kubernetes’ networking is configured under the hood.

## Network namespaces[](#network-namespaces)

Linux has a concept called [namespaces](https://en.wikipedia.org/wiki/Linux_namespaces). Namespaces are a feature that isolate the resources that a process sees from another processes. For example, a process may see MySQL running with PID 123 but a different process running in a different namespace (but on the same host) will see a different process assigned to PID 123, or none at all.

There are different kinds of namespaces; we are interested in the [Network (net)](https://en.wikipedia.org/wiki/Linux_namespaces#Network_(net)) namespace.

Each namespace has a virtual `loopback` interface and *may* have additional virtual network devices attached. Each of these virtual devices may be assigned exclusive or overlapping IP address ranges.

### localhost[](#localhost)

Processes running inside the same `net` namespace can send messages to each other over `localhost`.


[![pod-sandbox](https://llorllale.github.io/assets/img/k8s-networking/pod-sandbox.svg)](https://llorllale.github.io/assets/img/k8s-networking/pod-sandbox.svg "Traffic from a client to a server inside a network namespace. Blue is traffic on localhost. Notice the host’s interface (eth0) is bypassed entirely for this traffic.") *Traffic from a client to a server inside a network namespace. **Blue** is traffic on `localhost`. Notice the host’s interface (`eth0`) is bypassed entirely for this traffic.*

With this we have one or more processes that can communicate over `localhost`. This is exactly how K8s Pods work, and these “processes” are K8s containers.

## Connecting network namespaces on the same host[](#connecting-network-namespaces-on-the-same-host)

Remember that all pods in a K8s cluster can communicate with each other without NAT. So, how would two pods on the same host communicate with each other? Let’s give it a shot. Let’s create a “server” namespace and attempt to communicate with it.

[![disconnected-pods](https://llorllale.github.io/assets/img/k8s-networking/disconnected-pods.svg)](https://llorllale.github.io/assets/img/k8s-networking/disconnected-pods.svg)

We don’t have an address for `server` from within the `client` namespace yet. These two network namespaces are completely disconnected from each other. All `client` and `server` have is `localhost` (dev `lo`) which is always assigned `127.0.0.1`. We need another interface between these two namespaces for communication to happen.

Linux has the concept of *Virtual Ethernet Devices* ([veth](https://man7.org/linux/man-pages/man4/veth.4.html)) that act like “pipes” through which network packets flow, and of which you can attach either end to a namespace or a device. The “ends” of these “pipes” act as virtual devices to which IP addresses can be assigned. It is perfectly possible to create a *veth* device and connect our two namespaces like this:

[![pods-veth](https://llorllale.github.io/assets/img/k8s-networking/pods-veth.svg)](https://llorllale.github.io/assets/img/k8s-networking/pods-veth.svg)

However, consider that `veth` are *point-to-point* devices with just two ends and, remembering our requirement that all Pods must communicate with each other without NAT, we would need *veth* pairs, where is the number of namespaces. This becomes unwieldy pretty quickly. We will use a [bridge](https://wiki.linuxfoundation.org/networking/bridge) instead to solve this problem. A bridge lets us connect any number of devices to it and will happily route traffic between them, turning our architecture into a hub-and-spoke and reducing the number of *veth* pairs to just .

At this point the whole setup looks like this:

[![pods-bridge](https://llorllale.github.io/assets/img/k8s-networking/pods-bridge.svg)](https://llorllale.github.io/assets/img/k8s-networking/pods-bridge.svg "Two linux net namespaces connected to each other via a bridge. Note that although the bridge is connected to the host’s interface (eth0), traffic between the namespaces bypasses it entirely.") *Two linux `net` namespaces connected to each other via a bridge. Note that although the bridge is connected to the host’s interface (`eth0`), traffic between the namespaces bypasses it entirely.*

We have just connected two network namespaces on the same host.

## Connecting network namespaces on different hosts[](#connecting-network-namespaces-on-different-hosts)

The only way in and out of our hosts in our example above is via their `eth0` interface. For outbound traffic, the packets first need to reach `eth0` before being forwarded to the physical network. For inbound packets, `eth0` needs to forward those to the bridge where they will be routed to the respective namespace interfaces. Let’s first separate our two namespaces before going further.

### Moving our network namespaces onto different hosts[](#moving-our-network-namespaces-onto-different-hosts)

Let’s first clean up everything we’ve done so far[4](#fn:7):


Let’s now set up our namespaces in different hosts.

[![pod-different-hosts](https://llorllale.github.io/assets/img/k8s-networking/pods-diffhosts.svg)](https://llorllale.github.io/assets/img/k8s-networking/pods-diffhosts.svg "Namespaces on different hosts. The host interfaces (eth0) are on the same network.") *Namespaces on different hosts. The host interfaces (`eth0`) are on the same network.*

Now that everything is set up, let’s first tackle outbound traffic.

### From our network namespaces to the physical network[](#from-our-network-namespaces-to-the-physical-network)

First let’s see if we can reach `eth0` on each host:

The host isn’t reachable from the namespaces yet. *We haven’t configured an IP route[5](#fn:2) to forward packets destined to `eth0` in neither host.* Let’s set up a default route via the bridge in both namespaces and test:

Great, we can now reach our host interfaces. By extension, we can also reach any destination reachable from `eth0`:

This flow looks similar to the following when viewed from the `client` flow (Google’s infrastructure has been vastly simplified):

[![pods-outbound.svg](https://llorllale.github.io/assets/img/k8s-networking/pods-outbound.svg)](https://llorllale.github.io/assets/img/k8s-networking/pods-outbound.svg)

Next up, let’s try to communicate to our server from the `client` namespace.

### From the physical network to our network namespaces[](#from-the-physical-network-to-our-network-namespaces)

If we try to reach `server` from `client` we can see that it doesn’t work:

Let’s dig in with `tcpdump`.

Open a terminal window and, since we aren’t sure what path the packets are flowing through, run `tcpdump -nn -e -l -i any` on host `172.18.0.2`. **Friendly warning:** the output will be very verbose because `tcpdump` will listen on *all* interfaces.

On the same host `172.18.0.2`, try to curl the server from the `client` namespace again with `ip netns exec client curl -m 2 10.0.0.2:8080`. After it times out again, stop `tcpdump` by pressing `Ctrl+C` and review the output. Search for `10.0.0.2`, our destination address. You should spot some lines like the following:

`1 2  15:05:35.754605 bridge Out ifindex 5 a6:93:c7:0c:96:b2 ethertype ARP (0x0806), length 48: Request who-has 10.0.0.2 tell 10.0.0.0, length 28 15:05:35.754608 veth-clientbr Out ifindex 6 a6:93:c7:0c:96:b2 ethertype ARP (0x0806), length 48: Request who-has 10.0.0.2 tell 10.0.0.0, length 28`

You may see several of these requests with no corresponding reply[6](#fn:3).

These are [ARP](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) requests, and the reason they’re being fired off is that there is no \[IP ([layer 3](https://en.wikipedia.org/wiki/Network_layer))\] route between the `client` and `server` namespaces. It is possible to [manually configure ARP entries](https://www.xmodulo.com/how-to-add-or-remove-static-arp-entry-on-linux.html) and implement [“proxy-ARP”](https://tldp.org/HOWTO/Adv-Routing-HOWTO/lartc.bridging.proxy-arp.html) to connect `client` and `server` at [Layer 2](https://en.wikipedia.org/wiki/Data_link_layer), but we are not doing that today. Kubernetes’ networking model is built on Layer 3 and up, and so must our solution.

We will configure IP routing[5](#fn:2) rules to route `client` traffic to `server`. Let’s first configure a manual route for `10.0.0.2` on the client host:

`1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20  # on client host root@kind-control-plane:/# ip route add 10.0.0.2 via 172.18.0.4  # validate root@kind-control-plane:/# curl 10.0.0.2:8080 <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd"> <html> <head> <meta http-equiv="Content-Type" content="text/html; charset=utf-8"> <title>Directory listing for /</title> </head> <body> <h1>Directory listing for /</h1> <hr> <ul> ... </ul> <hr> </body> </html>`

As you can see, `curl`‘ing our server API in the `server` namespace from the client *host* now works[7](#fn:4).

Let’s try `curl`‘ing the server from the `client` *namespace* again:

Another dump with `tcpdump` reveals the same unanswered `ARP` requests as before. Why aren’t there responses to these considering we’ve successfully established a connection from the client *host* to the `server` namespace? One reason is that the connection was made at layer 3 (IP route), but `ARP` is a layer 2 protocol, and as per the [OSI model’s](https://en.wikipedia.org/wiki/OSI_model) semantics, lower-level protocols cannot depend on higher-level ones. Another reason is that `ARP` messages only reach devices directly connected to our network interface, in this case `eth0`: the latter’s `ARP` table does not contain an entry for `10.0.0.2` even though its namespace’s *IP routing* table does.

The layer 3 solution for us is simple: establish another IP route for `10.0.0.2` inside the `client` namespace[8](#fn:5):

`1  root@kind-control-plane:/# ip netns exec client ip route add 10.0.0.2 via 10.0.0.0`

You can now verify that calling `server` from `client` works:

**Congratulations 🎉 🎉** - we have just manually created two Pods (`net` namespaces) on different hosts, with one “container” (`curl`) in one Pod invoking an API in a container in the other Pod without NAT.

[![pod-pod-hosts](https://llorllale.github.io/assets/img/k8s-networking/pod2pod-diff-hosts.svg)](https://llorllale.github.io/assets/img/k8s-networking/pod2pod-diff-hosts.svg "A process inside a client namespace connecting to an open socket on a server namespace in another host. The client process does not perform any NAT.") *A process inside a `client` namespace connecting to an open socket on a `server` namespace in another host. The client process does not perform any NAT.*

## How Kubernetes creates Pods[](#how-kubernetes-creates-pods)

We now know how pods are implemented under the hood. We have learned that Kubernetes “pods” are namespaces and that Kubernetes “containers” are processes running within those namespaces. These pods are connected to each other within each host with virtual networking devices (`veth`, `bridge`), and with simple IP routing rules for traffic to cross from one pod to another over the physical network.

Where and how does Kubernetes do all this?

### The Container Runtime Interface (CRI)[](#the-container-runtime-interface-cri)

Back in the [concepts section](#concepts) we said the `kubelet` uses the [Container Runtime Interface](https://github.com/kubernetes/cri-api) to create the pod “sandboxes”.

The `kubelet` creates pod sandboxes [here](https://github.com/kubernetes/kubernetes/blob/67b38ffe6ea3350f3cefd72caacd3f7ee9b1af42/pkg/kubelet/kuberuntime/kuberuntime_sandbox.go#L68-L73). Note that `runtimeService` is of type [`RuntimeService`](https://github.com/kubernetes/cri-api/blob/adbbc6d75b383d6b823c24bba946029458d6681b/pkg/apis/services.go#L106-L118), belonging to the CRI API. It embeds the `PodSandboxManager` type, which is responsible for actually creating the sandboxes (`RunPodSandbox` method). Kubernetes has an internal implementation of `RuntimeService` in [`remoteRuntimeService`](https://github.com/kubernetes/kubernetes/blob/805be30745defc72cb6137a25b3e821db4056837/pkg/kubelet/cri/remote/remote_runtime.go#L45-L52), but this is just a thin wrapper around the CRI API’s [`RuntimeServiceClient`](https://github.com/kubernetes/cri-api/blob/adbbc6d75b383d6b823c24bba946029458d6681b/pkg/apis/runtime/v1/api.pb.go#L10076-L10168) (GitHub won’t automatically open the file due to its size). Look closely and you’ll notice that `RuntimeServiceClient` is implemented by [`runtimeServiceClient`](https://github.com/kubernetes/cri-api/blob/adbbc6d75b383d6b823c24bba946029458d6681b/pkg/apis/runtime/v1/api.pb.go#L10170-L10172), which uses a [gRPC](https://grpc.io/) connection to invoke the container runtime service. gRPC is (normally) transported over TCP sockets ([Layer 3](https://en.wikipedia.org/wiki/Transport_layer)).

The `kubelet` runs on each node and, if it needs to create a pod on that node, why would it need to communicate with the CRI service over TCP?

Go, the *lingua franca* of cloud-native development (including Kubernetes), has a builtin [`plugin`](https://pkg.go.dev/plugin) system but it has some serious drawbacks in terms of maintainability. Eli Bendersky gives a good outline of how they work with pros and cons [here](https://eli.thegreenplace.net/2021/plugins-in-go/) that is worth a read. Towards the end of the article you’ll notice a bias towards RPC-based plugins; this is exactly what the CRI’s designers chose as their architecture. So although the `kubelet` and the CRI service are running on the same node, the gRPC messages can be transported locally via `localhost` (for TCP) or [Unix domain sockets](https://en.wikipedia.org/wiki/Unix_domain_socket) or some other channel available on the host.

So we now have Kubernetes invoking the standard CRI API that in turn invokes a “remote”, CRI-compliant gRPC service. This service is the CRI implementation that can be swapped out. Kubernetes’ docs list [a few common ones](https://kubernetes.io/docs/setup/production-environment/container-runtimes/):

-   [containerd](https://github.com/containerd/containerd)
-   [CRI-O](https://github.com/cri-o/cri-o)
-   [Docker Engine](https://github.com/moby/moby)
-   [Mirantis Container Runtime](https://github.com/Mirantis/cri-dockerd)

The details of what happens next vary by implementation, and is all abstracted away from the Kubernetes runtime. Take `containerd` as an example (it’s the CRI used in [kind](https://github.com/kubernetes-sigs/kind), the K8S distribution I chose for the [walkthrough](#create-your-own-pod-network) above). `containerd` has a plugin architecture that is resolved at compile time[9](#fn:8). `containerd`’s [implementation](https://github.com/containerd/containerd/blob/a338abc902d9f204dcb9df7212d39fd7d07ac06d/pkg/cri/server/service.go#L78-L128) of `RuntimeServiceServer` (part of [Concepts](#concepts)) has its [`RunPodSandbox`](https://github.com/containerd/containerd/blob/3ee6dd5c1bca441d1ec4988cbaebadbfbcfde525/pkg/cri/server/sandbox_run.go#L56-L407) method (also part of [Concepts](#concepts)) rely on a “CNI” plugin to set up the pod’s network namespace.

What is the CNI?

### The Container Network Interface (CNI)[](#the-container-network-interface-cni)

The [CNI](https://github.com/containernetworking/cni) is used by the CRI to create and configure the network namespaces used by the pods[10](#fn:9). CNI implementations are invoked by executing their respective binaries and providing network configuration via `stdin` (see the spec’s [execution protocol](https://github.com/containernetworking/cni/blob/main/SPEC.md#section-2-execution-protocol))[11](#fn:10). On unix hosts, `containerd` by default looks for a standard CNI config file inside the `/etc/cni/net.d` directory and for the plugin binaries it looks in `/opt/cni/bin` (see [code](https://github.com/containerd/containerd/blob/3bc8fc4d3067c32d2580e716af095a837c0fbe9a/pkg/cri/config/config_unix.go#L68-L69)). Each node in my `kind` cluster has only one config file: `/etc/cni/net.d/10-kindnet.conflist`. Here are the contents of this file in my `control-plane` node:

Click to expand

The same config file on the worker nodes have identical content except for `subnet`, which varies from host to host. I won’t go in depth about how the CNI spec and plugins work (that deserves its own article). You can read version `0.3.1` of the spec [here](https://github.com/containernetworking/cni/blob/spec-v0.3.1/SPEC.md). What’s conceptually important for us is that there are three plugins being executed (two of them are chained) with this configuration. These plugins are:

-   [ptp](https://www.cni.dev/plugins/current/main/ptp/): creates a point-to-point link between a container and the host by using a veth device.
-   [host-local](https://www.cni.dev/plugins/current/ipam/host-local/): allocates IPv4 and IPv6 addresses out of a specified address range.
-   [portmap](https://www.cni.dev/plugins/current/meta/portmap/): will forward traffic from one or more ports on the host to the container.

Do any of these concepts sound familiar to you? They should! These are the things we painstakingly configured step-by-step in our walkthrough above. With this information in mind, go back to the component diagram in [Concepts](#concepts) and map each of these concepts to the boxes in the diagram.

No discussion of Kubernetes’ cluster network can conclude without mentioning [Services](https://kubernetes.io/docs/concepts/services-networking/service/).

Conceptually, a Kubernetes *Service* is merely a [Virtual IP](https://en.wikipedia.org/wiki/Virtual_IP_address) assigned to a set of pods, and to which a stable [DNS name](https://en.wikipedia.org/wiki/Domain_Name_System) is assigned. Kubernetes also provides simple load balancing out of the box for some types of services (`ClusterIP`, `NodePort`).

Each service is mapped to a set of IPs belonging to the pods exposed by the service. These set of IPs is called [EndpointSlice](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/) and is constantly updated to reflect the IPs currently in use by the backend pods[13](#fn:13). Which pods? The ones matching the service’s *selector*.

Example Service with label ‘myLabel’ set to value ‘MyApp’

When a user creates a new Service:

1.  `kube-apiserver` assigns it the next free IP by incrementing a counter stored in `etcd`[14](#fn:16).
2.  `kube-apiserver` stores the service in `etcd`[15](#fn:17).
3.  This event is pushed to all [watches](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes)[16](#fn:14).
4.  `coreDNS`:
    1.  Event is caught and the service’s name, namespace, and (virtual) cluster IP is cached[17](#fn:18).
    2.  Responds to requests for A records by reading from the cache[18](#fn:19).
5.  [EndpointSlice](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/) Controller: event is caught and a new EndpointSlice is assigned to the service[19](#fn:20).
6.  `kube-proxy`: event is caught and `iptables` is configured on worker nodes[20](#fn:21).

All steps from 4 onwards are executing concurrently by independent processes. The final state is depicted in the diagram in the [Concepts](#concepts) section.

Note that we have incidentally glossed over Kubernetes’ distributed and event-driven architecture. We’ll expand on that topic in a future article.

We snuck in a new concept in step 6: `iptables`. Let’s expand on that next.

## iptables[](#iptables)

> Iptables is used to set up, maintain, and inspect the tables of IP packet filter rules in the Linux kernel. Several different tables may be defined. Each table contains a number of built-in chains and may also contain user-defined chains.
> 
> Each chain is a list of rules which can match a set of packets. Each rule specifies what to do with a packet that matches. This is called a \`target’, which may be a jump to a user-defined chain in the same table.
> 
> – `iptables` manpage

System and network administrators use `iptables` to configure IP routing rules on *Linux* hosts, and so does `kube-proxy`[21](#fn:15). On *Windows* hosts `kube-proxy` uses an analogous API called [Host Compute Network service API](https://learn.microsoft.com/en-us/windows-server/networking/technologies/hcn/hcn-top), internally represented by the [HostNetworkService](https://github.com/kubernetes/kubernetes/blob/5eb6f82c1ade7ceac0e9f26283d35ec806e47b9f/pkg/proxy/winkernel/hns.go#L33-L44) interface. It is because of this difference in OS-dependent implementations of the network stack that we simply labelled them as “OS IP rules” in the [Concepts](#concepts) section’s diagram.

`kube-proxy` uses `iptables` to configure Linux hosts to distribute traffic directed at a Service’s `clusterIP` (ie. a *virtual* IP) to the backend pods selected by the service using [NAT](https://en.wikipedia.org/wiki/Network_address_translation). So yes, there is definitely network address translation in a Kubernetes cluster, but it’s hidden from your workloads.

`kube-proxy` adds a rule to the `PREROUTING` chain that targets a custom chain called `KUBE-SERVICES`[22](#fn:22). The end result looks like this:

Initially the `KUBE-SERVICES` chain contains rules just for the `NodePort` custom chain and several built-in services:


New rules are appended for each service by the Proxier’s `syncProxyRules` method and are written [here](https://github.com/kubernetes/kubernetes/blob/b9bc0e5ac8032bb63298a407c287e6055ef073de/pkg/proxy/iptables/proxier.go#L1095-L1103). For example, the following shows a rule targeting a custom chain `KUBE-SVC-BM6F4AVTDKG47F3K` for a service named `mysvc`:

If we inspect `KUBE-SVC-BM6F4AVTDKG47F3K` we see something interesting:

Ignoring the masq for now, we see three rules targeting chains for *service endpoints*. `kube-proxy` adds these entries as it handles incoming events for endpointslices (see [NewProxier()](https://github.com/kubernetes/kubernetes/blob/b9bc0e5ac8032bb63298a407c287e6055ef073de/pkg/proxy/iptables/proxier.go#L267)). Each rule has a helpful comment indicating the target service endpoint.

Note how these rules have a probability assigned to them. Rules in `iptables` chains are processed sequentially. In this example there are three *service endpoint* rules, and the first is assigned a probability of `0.33`. Next, if the dice roll failed on the first one, we roll it again for the second rule, this time with a probability of 50%. If that fails, we fall back to the third rule with a probability of 100%. In this way we have an even distribution of traffic amongst the three endpoints. The probabilities are set [here](https://github.com/kubernetes/kubernetes/blob/b9bc0e5ac8032bb63298a407c287e6055ef073de/pkg/proxy/iptables/proxier.go#L1635-L1641). Note how the probability curve is fixed as a flat distribution, and also note how `kube-proxy` is not balancing this traffic itself. As noted in [Concepts](#concepts), `kube-proxy` is not itself in the *data plane*.

In our example above, `mysvc` is selecting three pods with endpoints `10.244.1.2:8080`, `10.244.2.2:8080`, and `10.244.2.3:8080`.

This is the service definition:

And these are the IPs assigned to the selected pods (take note of the nodes as well):

If we inspect one of the service endpoint chains we see something else interesting:

We see a `DNAT` (*destination* NAT) rule that *translates* the destination address to `10.244.1.2:8080`. We already know that this destination is hosted on node `kind-worker`, so investigating on that node we see:
																																																																																																																																																																																																																																																																																																																																																																																																																										   

We are back in `net` namespace land!

In our case, we are running nginx on a simple deployment:

Spec

Kubernetes is an event-driven, distributed platform that automates the deployment and networking aspects of your workloads. `kube-apiserver` is the platform’s “event hub”.

[![k8s-create-pod-service](https://llorllale.github.io/assets/img/k8s-networking/k8s-create-pod-service.svg)](https://llorllale.github.io/assets/img/k8s-networking/k8s-create-pod-service.svg "Blue arrows show where configuration data for Deployments flow. Red arrows show where configuration data for Services flow. Note that this is just a subset of all the machinery activated when a user creates either of these two resources.") *Blue arrows show where configuration data for Deployments flow. Red arrows show where configuration data for Services flow. Note that this is just a subset of all the machinery activated when a user creates either of these two resources.*

`kubelet` runs on each node and listens for events from `kube-apiserver` where pods are added to the node it’s running on. When a pod is created, be it with a controller or just an orphaned pod, `kubelet` uses the Container Runtime Interface (CRI) to create the pod’s sandbox. The CRI in turn uses the Container Network Interface to configure the pod’s network namespace on the node. The pod will have an IP that is reachable by any other pod in any other node.

When a `ClusterIP` Service is created, `kube-apiserver` assigns a free *Virtual IP* to it and persists the Service object to `etcd`. The event is caught by `coreDNS` which proceeds to cache the service\_name -> cluster\_ip mapping, and respond to DNS requests accordingly. The event is also caught by the EndpointSlice controller which then creates and attaches an EndpointSlice with the IPs of the selected Pods to the Service and saves the update to `etcd`.

`kube-proxy` runs on each node and listens for events from `kube-apiserver` where Services and EndpointSlices are added and configures the local node’s IP routing rules to point the Service’s virtual IP to the backend Pods with an even distribution.

During runtime, a client container queries `coreDNS` for the Service’s address and directs its request to the Service’s virtual IP. The local routing rules (`iptables` on Linux hosts, `Host Compute Service API` on Windows) randomly select one of the backend Pod IP addresses and forwards traffic to that Pod.

---

**Footnotes**