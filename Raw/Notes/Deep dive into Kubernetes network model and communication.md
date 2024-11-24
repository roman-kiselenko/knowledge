---
title: Deep dive into Kubernetes network model and communication
source: https://addozhang.medium.com/deep-dive-into-kubernetes-network-model-and-communication-57a2bffc852e
clipped: 2024-02-05
published: 
category: network
tags:
  - k8s
read: false
---

This is the first note on Kubernetes network learning.

In this series, we will explore the Kubernetes network model and communication through the following articles:

-   [Deep Dive into Kubernetes Network Model and Communication](https://medium.com/@addozhang/deep-dive-into-kubernetes-network-model-and-communication-57a2bffc852e)
-   [Understanding the Container Network Interface (CNI)](https://addozhang.medium.com/introduction-to-container-network-interface-cni-25309a64b23e)
-   [Source code analysis: how kubelet and container runtime work with CNI](https://medium.com/@addozhang/source-code-analysis-understanding-cnis-usage-from-kubelet-container-runtime-24d72f29466b)
-   [Learning Kubernetes VXLAN network from Flannel](https://addozhang.medium.com/learning-kubernetes-vxlan-network-with-flannel-2d6a58c95300)
-   [Kubernetes network learning with Cilium and eBPF](https://addozhang.medium.com/kubernetes-network-learning-with-cilium-and-ebpf-aafbf3163840)

Kubernetes defines a simple and consistent network model based on a flat network structure. This design allows for efficient communication without the need to map host ports to network ports or use other forwarding components. The model also makes it easy for applications to migrate from virtual machines or physical machines to pods managed by Kubernetes.

In this article, we will dive into the Kubernetes network model and understand how communication takes place between containers and pods. The implementation of the network model will be covered in later articles.

![[Raw/Media/Resources/4589676133b5d578299ff9ab2e4c4441_MD5.png]]

The Kubernetes network model defines the following:

-   Each pod has its own IP address, which is reachable within the cluster.
-   All containers within a pod share the same IP address (including MAC address) and can communicate with each other using `localhost`.
-   Pods can communicate with any other pod in the cluster using the pod IP address without the need for NAT.
-   Kubernetes components can communicate with each other and with pods.
-   Network isolation can be achieved through network policies.

The definition above mentions several related components:

-   Pod: In Kubernetes, a pod is similar to a virtual machine with a unique IP address. Pods on the same node share network and storage.
-   Container: A pod is a collection of containers that share the same network namespace. Containers within a pod communicate with each other using `localhost`. Containers have their own independent file system, CPU, memory, and process space. Containers are created by creating a pod.
-   Node: Pods run on nodes, and a cluster can have one or more nodes. The network namespace of each pod is connected to the namespace of the node to establish connectivity.

After discussing the network namespace so many times, how does it actually work?

In the Kubernetes distribution [k3s](https://k3s.io/), a pod is created with two containers: a `curl` container for sending requests and an `httpbin` container for providing web services.

*Although k3s is a distribution, it still uses the Kubernetes network model, which does not prevent us from understanding the network model.*

```yaml
apiVersion: v1  
kind: Pod  
metadata:  
  name: multi-container-pod  
spec:  
  containers:  
  - image: curlimages/curl  
    name: curl  
    command: ["sleep", "365d"]  
  - image: kennethreitz/httpbin  
    name: httpbin
```

Log in to the node and use `lsns -t net` to list the network namespaces on the current host. However, we cannot find the process of `httpbin`. There is a command for the namespace called `/pause`. This `pause` process is actually an *invisible* sandbox container process in each pod. The role of the sandbox container will be introduced in the next article on container network and CNI.

```
lsns -t net  
        NS TYPE NPROCS    PID USER     NETNSID NSFS                                                COMMAND  
4026531992 net     126      1 root  unassigned                                                     /lib/systemd/systemd --system --deserialize 31  
4026532247 net       1  83224 uuidd unassigned                                                     /usr/sbin/uuidd --socket-activation  
4026532317 net       4 129820 65535          0 /run/netns/cni-607c5530-b6d8-ba57-420e-a467d7b10c56 /pause
```

Since each container has its own process namespace, let’s switch to the process type namespace to see the process type space:

```
lsns -t pid  
        NS TYPE NPROCS    PID USER            COMMAND  
4026531836 pid     127      1 root            /lib/systemd/systemd --system --deserialize 31  
4026532387 pid       1 129820 65535           /pause  
4026532389 pid       1 129855 systemd-network sleep 365d  
4026532391 pid       2 129889 root            /usr/bin/python3 /usr/local/bin/gunicorn -b 0.0.0.0:80 httpbin:app -k gevent
```

With the process PID `129889`, we can find the namespace it belongs to:

```
ip netns identify 129889  
cni-607c5530-b6d8-ba57-420e-a467d7b10c56
```

Then we can use `exec` to run commands in that namespace:

```
ip netns exec cni-607c5530-b6d8-ba57-420e-a467d7b10c56 ip a  
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000  
    linkgloopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
    inet 127.0.0.1g8 scope host lo  
       valid_lft forever preferred_lft forever  
    inet6 ::1g128 scope host  
       valid_lft forever preferred_lft forever  
2: eth0@if17: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default  
    linkgether f2:c8:17:b6:5f:e5 brd ff:ff:ff:ff:ff:ff link-netnsid 0  
    inet 10.42.1.14/24 brd 10.42.1.255 scope global eth0  
       valid_lft forever preferred_lft forever  
    inet6 fe80::f0c8:17ff:feb6:5fe5g64 scope link  
       valid_lft forever preferred_lft forever
```

From the result, we can see that the IP address of the pod `10.42.1.14` is bound to the interface eth0, and eth0 is connected to the interface 17.

On the node host, we can check the information of interface 17. `veth7912056b` is a virtual Ethernet interface (vitual ethernet device) in the host root namespace, which is a tunnel connecting the pod network and the node network, with the other end being the interface eth0 in the pod namespace.

```
ip link | grep -A1 ^17  
17: veth7912056b@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master cni0 state UP mode DEFAULT group default  
    link/ether d6:5e:54:7f:df:af brd ff:ff:ff:ff:ff:ff link-netns cni-607c5530-b6d8-ba57-420e-a467d7b10c56
```

From the previous result, we can see that this `veth` is connected to a network bridge `cni0`.

> *A network bridge works at the data link layer (Layer 2 of the OSI model), connecting multiple networks (or network segments). When a request arrives at the bridge, the bridge asks all connected interfaces (here, the pod is connected to the bridge through the veth interface) if they have the IP address in the original request. If an interface responds, the bridge records the matching information (IP -> veth) and forwards the data.*

But what happens if there is no interface response? The specific process depends on the implementation of various network plugins. I plan to introduce commonly used network plugins, such as Calico, Flannel, Cilium, etc., in later articles.

Next, let’s take a look at how network communication is completed in Kubernetes. There are several types:

-   Communication between containers within the same pod
-   Communication between pods on the same node
-   Communication between pods on different nodes

Communication between containers in the same pod is simple. These containers share a network namespace, and each namespace has a `lo` loopback interface, which can be used to communicate via `localhost`.

![[Raw/Media/Resources/3f5530c73695c4c399ed7b53bf782621_MD5.png]]

When we run the `curl` container and the `httpbin` container in two separate pods, they may be scheduled to the same node. The request sent by `curl` reaches the `eth0` interface of the pod according to the container's routing table. Then, it reaches the node's root network namespace through the `veth1` tunnel connected to `eth0`.

`veth1` is connected to other pods through the virtual Ethernet interface `vethX` connected to the bridge `cni0`. The bridge will ask all connected interfaces if they have the IP address in the original request (such as `10.42.1.9` here). After receiving a response, the bridge records the mapping information (`10.42.1.9` => `veth0`) and forwards the data. Eventually, the data enters the `httpbin` pod through the `veth0` tunnel.

![[Raw/Media/Resources/1b0d71d4b23ecdbdf9d25c8fcd308392_MD5.png]]

Communication between pods across nodes is more complex, and **different network plugins handle it differently**. Here, we choose an easy-to-understand way to briefly explain it.

The first part of the process is similar to communication between pods on the same node. When the request reaches the bridge, the bridge asks which pod has the IP address but does not receive a response. The process enters the host’s routing addressing process and moves to a higher level in the cluster.

There is a routing table at the cluster level that stores the Pod IP subnet of each node. When a node joins the cluster, it is assigned a Pod subnet (Pod CIDR). For example, the default Pod CIDR in k3s is `10.42.0.0/16`, and the node obtains the subnet of `10.42.0.0/24`, `10.42.1.0/24`, `10.42.2.0/24`, and so on, in turn. The node that should receive the request can be determined by its Pod IP subnet, and the request is sent to that node.

![[Raw/Media/Resources/0df0aed1159732f33103d1e1a515d9b9_MD5.png]]

Now, you should have a basic understanding of Kubernetes networking.

The entire communication process requires the coordination of various components, such as the pod network namespace, the pod Ethernet interface `eth0`, virtual Ethernet interfaces `vethX`, network bridge `cni0`, and so on. Some of these components correspond to pods one-to-one and have the same lifecycle as the pods. Although they can be created, associated, and deleted manually, it is not realistic for non-permanent resources like pods, which are frequently created and destroyed, to rely on too much manual work.

In fact, these tasks are delegated to network plugins by the container, which follow the CNI (Container Network Interface) specification.

What do network plugins do?

-   Create the network namespace for the pod (container)
-   Create interfaces
-   Create veth pairs
-   Set namespace networking
-   Set static routes
-   Configure Ethernet bridge
-   Assign IP addresses
-   Create NAT rules
-   …

-   [https://www.tigera.io/learn/guides/kubernetes-networking/](https://www.tigera.io/learn/guides/kubernetes-networking/)
-   [https://kubernetes.io/docs/concepts/services-networking/](https://kubernetes.io/docs/concepts/services-networking/)
-   [https://matthewpalmer.net/kubernetes-app-developer/articles/kubernetes-networking-guide-beginners.html](https://matthewpalmer.net/kubernetes-app-developer/articles/kubernetes-networking-guide-beginners.html)
-   [https://learnk8s.io/kubernetes-network-packets](https://learnk8s.io/kubernetes-network-packets)