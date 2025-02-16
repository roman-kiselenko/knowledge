---
title: How to traceroute Kubernetes pod-to-pod traffic
source: https://medium.com/globant/tracerouting-pod-to-pod-traffic-a45fabd86f77
clipped: 2023-09-29
published: 
category: network
tags:
  - pod
  - network
read: false
---

Welcome to this article on *pod-to-pod* communication in Kubernetes! As you embark on this journey, we will explore the intricacies of Kubernetes networking and delve into the fundamental principles and mechanisms that govern pod-to-pod communication.

In this article, we will focus on the Kubernetes networking model within the context of VirtualBox’s default networking layout. By examining this setup, we will gain a solid understanding of how pod-to-pod communication works in a Kubernetes environment. We will uncover the essential concepts and mechanisms that enable seamless communication between pods within a cluster.

Whether you’re seeking a comprehensive introduction to Kubernetes networking or looking to optimize communication within your projects, this article will provide valuable insights and practical knowledge. Join us as we unravel the essentials of efficient pod-to-pod communication in Kubernetes. Let’s dive in and explore the fascinating world of Kubernetes networking together!

In the realm of Kubernetes networking, it is essential to note that it does not come bundled with a native networking implementation. Instead, it adopts a flexible approach by decoupling the cluster networking implementation using the [Container Runtime Interface](https://github.com/kubernetes/cri-api) (CRI) and [Container Network Interface](https://github.com/containernetworking/cni) (CNI) standards. These standards serve as gateways for third-party providers to offer their own Kubernetes networking implementation plugins, which in turn offer a multitude of options to choose from when setting up a Kubernetes cluster.

Given this diversity, it becomes apparent that there isn’t a single prescribed networking solution but rather a rich array of alternatives available to Kubernetes users. Each option within this extensive repertoire comes with its own set of distinctive characteristics and intricacies, which can significantly impact the resulting cluster networking implementation. This broad spectrum of choices ensures that organizations can select a networking approach that best aligns with their specific requirements, considering scalability, performance, security, and compatibility with existing infrastructure.

It is worth emphasizing that diverse Kubernetes networking options empower organizations to fine-tune their cluster’s networking configuration to meet their unique needs. However, this wealth of choices can also present challenges, as the selection process demands careful evaluation of the trade-offs and considerations associated with each networking solution. Organizations can make informed decisions that facilitate optimal performance, reliable communication, and streamlined management within their Kubernetes clusters by comprehending the nuances of different Kubernetes networking options.

When it comes to Kubernetes CRI and CNI plugins, you have a range of popular options. On the CRI side, [*containerd*](https://github.com/containerd/containerd) and [*cri-o*](https://github.com/cri-o/cri-o) emerge as two popular choices, while on the CNI side, [*Calico*](https://github.com/projectcalico/calico) and [*Flannel*](https://github.com/flannel-io/flannel) stand out as popular alternatives. These plugins provide essential functionality and features for container runtime and network management within Kubernetes clusters, allowing you to select the options that best align with your specific requirements and preferences.

A functional Kubernetes cluster requires the CRI and CNI plugins to coexist harmoniously. Such coexistence between the plugins demands configuration efforts to ensure they work as desired. Therefore, Kubernetes distributions emerged as an alternative way to make Kubernetes usage easy for the masses.

A Kubernetes distribution installs and configures all Kubernetes components, including the CRI and CNI plugins. You do not need to worry about the low-level platform configuration details. They are all gathered for you to jump right into Kubernetes applications. All you need to do is run a command, wait a couple of minutes for the platform to deploy all resources, and access the Kubernetes API provided to you once the deployment process finishes.

You can find commercial distributions like Red Hat Openshift, AWS EKS, Google GKS, and open-source distributions like OKD and minikube.

In this article, you explore pod-to-pod traffic using [vagrant-k8s-lab](https://gitlab.com/areguera/vagrant-k8s-lab). This small open-source Kubernetes distribution initiates the cluster locally, using [Vagrant](https://www.vagrantup.com/) with VirtualBox provider and Ansible roles for provisioning the cluster machines. [vagrant-k8-lab](https://gitlab.com/areguera/vagrant-k8s-lab) exists for studying purposes only, and it happily accepts contributions from anyone.

The [vagrant-k8-lab](https://gitlab.com/areguera/vagrant-k8s-lab) project installs and configures containerd CRI and flannel CNI plugins in all cluster nodes. The cluster architecture looks like the following:

![[Raw/Media/Resources/b2bf10700c57a4285c63c67b9cec752c_MD5.png]]

**Figure 1.** Kubernetes cluster and network interfaces.

The “*Figure 1.* Kubernetes cluster and network interfaces,” describes the cluster architecture you use in this article about pod-to-pod communication. Note it has four network interfaces attached to each node. The network interfaces are *lo, eth0, eth1,* and *cni0*. Each interface has its purpose and reason to exist. Additionally, the cluster has a couple of pods already deployed for testing purposes later in this article.

The CRI plugin needs outbound traffic in a Kubernetes cluster to download container images from external container registries. The operating system package manager must also reach remote package repositories to download system updates. These two actions demand Kubernetes nodes to have outbound traffic.

For example, when the Kubernetes Scheduler commands the Kubelet running in a node to deploy a pod, it is that node’s CRI plugin's responsibility to contact the external container registries to download the container image defined in the pod manifest the Kubernetes Scheduler commanded to deploy. In case that node cannot communicate with the external container registry, the pod creation fails.

When you deploy Kubernetes nodes using VirtualBox virtual machines, they are created with the *eth0* network interface already attached in NAT mode. At the IP level, the guest operating system receives the 10.0.2.15/24 address and can communicate with VirtualBox through the 10.0.2.3/24 address. This networking layout is known as VirtualBox default networking layout. It allows virtual machines to communicate with external resources as long as the host running VirtualBox has the required connectivity.

![[Raw/Media/Resources/cf558d31eedb42358697d43fbd3183a4_MD5.png]]

**Figure 2.** VirtualBox default networking layout.

VirtualBox default networking layout may look counterintuitive until you realize the virtual machines are connected to different networks, even though they all use the same address space. This VirtualBox design choice allows external world communication using a consistent IP addressing schema on all virtual machines.

In VirtualBox's default networking layout, the connections between virtual machines are treated individually. Each virtual machine is connected to its own isolated “*10.0.2.0/24*” network inside VirtualBox, probably using a unique virtual link for each one internally. Consequently, virtual machines can communicate with the external world but cannot communicate between themselves.

## Practical example:

1\. Create a pod named nginx using the nginx image:

\[vagrant@node-1 ~\]$ kubectl run nginx --image=nginx:latest

2\. Check the pod status:

\[vagrant@node-1 ~\]$ kubectl get pods/nginx  
NAME    READY   STATUS    RESTARTS   AGE  
nginx   1/1     Running   0          10m

The pod status is *Running*. Nice! This action required *containerd* to contact a remote container registry, like registry.docker.io, and download the nginx container image. If the pod is running, it means those actions took place successfully in node-1, assuming node-1 was the node Kubernetes Scheduler chose to run the actions. You can have additional information about these actions by running the following commands:

\[vagrant@node-1 ~\]$ kubectl describe pods/nginx

\[vagrant@node-1 ~\]$ journalctl -f -u containerd

The Flannel CNI plugin, when configured with *host-gw* backend, requires the Kubernetes nodes to have direct connectivity between them—for example, connecting each node to the same physical ethernet link. The *eth0* network interface does not satisfy such a need because it uses VirtualBox’s *NAT Network* type. To satisfy the Flannel CNI plugin’s necessities, it is necessary to attach a different network type to all nodes in the Kubernetes cluster.

VirtualBox has different network types you can choose from when creating new networks. The Kubernetes nodes use the *“Host-only networking”* type in this article. Quoting VirtualBox’s documentation:

*“This can be used to create a network containing the host and a set of virtual machines, without the need for the host’s physical network interface. Instead, a virtual network interface, similar to a loopback interface, is created on the host, providing connectivity among virtual machines and the host.”*

VirtualBox’s host-only networking type allows communication between nodes inside the cluster. By default, the Flannel CNI plugin configuration uses an eth0 network interface for node-to-node communication. [The cluster installation process this article used](https://gitlab.com/areguera/vagrant-k8s-lab) changed the default CNI plugin configuration to refer to eth1 instead, the new network interface value used for *inter-node* communication.

Here is a visual representation of the cluster with the eth1 network attached:

![[Raw/Media/Resources/b80371e42a6cef18cbf5b5de8b864f06_MD5.png]]

**Figure 3.** Attach eth1 to VirtualBox default network layout.

In “*Figure 3*. Attach eth1 to VirtualBox default network layout,” the inter-node communication happens through the *192.168.56.0/24* network address. Each node has an IP address assigned in this range. Starting at 192.168.56.10/24 for the control plane, then 192.168.56.11/24 for the first worker node, and 192.168.56.12/24 for the second worker node.

## Practical example:

1\. Check connectivity from the control-plane node to node-1 worker:

\[vagrant@controlplane ~\]$ traceroute -n 192.168.56.11  
traceroute to 192.168.56.11 (192.168.56.11), 30 hops max, 60 byte packets  
 1 192.168.56.11 0.900 ms 0.850 ms 0.717 ms

2\. Check connectivity from the control-plane node to node-2 worker:

\[vagrant@controlplane ~\]$ traceroute -n 192.168.56.12  
traceroute to 192.168.56.12 (192.168.56.12), 30 hops max, 60 byte packets  
 1 192.168.56.12 1.003 ms 0.837 ms 0.851 ms

Based on the practical verification, node-1 and node-2 are reachable from the control plane. Excellent!

The pod-to-pod traffic happens when you run Kubernetes applications in a single node. In such a scenario, pods must be able to share information and have IP addresses that allow permanent communication. Remember, pods are temporary; they can be destroyed and created anytime. The networking logic that makes this possible is mainly managed by the CNI plugin you’ve installed and configured in the cluster.

Let’s visualize the architecture of node-1, where two pods are running:

![[Raw/Media/Resources/fade5001a882ea1d493e17f0236b1a0a_MD5.png]]

**Figure 4.** Network interfaces attached to node-1.

In “*Figure 4, Network interfaces attached to node-1,”* the *cni0* network interface connects all pods in node-1, the first node we attached to the cluster, after creating the control-plane node. Through the *cni0* network interface, the CNI plugin manages the assignment of IP addresses of Kubernetes network resources, such as pods. Whenever you create a network resource, it receives a dynamic IP address, and when the resource is destroyed, the IP address assigned is released for reassignment later. To preserve the relationship between assigned and unassigned IP addresses, the Flannel CNI plugin uses [*etcd*](https://etcd.io/), the Kubernetes Storage service.

## Practical example:

1.  Create one pod named httpd using the httpd image.

\[vagrant@node-1 ~\]$ kubectl run httpd --image=httpd

2\. Check the httpd pod status to get its IP address. Let’s assume it is 10.244.1.3/24:

\[vagrant@node-1 ~\]$ kubectl get pods -o wide

3\. Create a new pod using the busybox image to run a traceroute command against the httpd pod IP address:

\[vagrant@node-1 ~\]$ kubectl run traceroute --image=busybox -- traceroute 10.244.1.3

4\. Check the logs output in the traceroute pod:

\[vagrant@node-1 ~\]$ kubectl logs traceroute  
traceroute to 10.244.1.3 (10.244.1.3), 30 hops max, 80 byte packets  
 1  10.244.1.3 (10.244.1.3)  0.018 ms  0.009 ms  0.005 ms

The traceroute logs show *one single hop* to reach the target pod, which is expected, considering both are in the same 10.244.1.0/24 network. With the evidence, we can conclude that *the communication happens locally through the cni0 network virtual interface when pods are running in the same node. In these cases, the network traffic never leaves the node*.

A Kubernetes cluster with a single node is useful for some conceptual tests but doesn’t provide redundancy for your workloads. If the node fails, your application fails. Redundancy is the result of running your application in a multi-node cluster. When running a multi-node cluster, understanding how the traffic flows from one resource to another gives you a powerful tool during troubleshooting. Such understanding depends significantly on the CNI plugin configured in your Kubernetes cluster since each CNI plugin has its approach to solving this problem.

Nodes need to know where to send traffic inside the cluster. To solve this problem, when configured with the host-gw backend, the Flannel plugin uses the operating system route table to manage that kind of knowledge on each node. All this work is covered by the CNI plugin automatically when you add new nodes to or remove old ones from the cluster. You don’t need to maintain the route tables yourself. The CNI plugin does it for you. When you think of it, that is a lot of work!

Let’s take our Kubernetes cluster visualization a step further and consider three nodes, each one running two pods, in 3 different CNI networks connected through the eth1 network and network routes already set for the operating system to know where to send packages inside the cluster:

![[Raw/Media/Resources/0296619baba44962627556033c9748bc_MD5.png]]

**Figure 5.** Multi-node cluster networking.

Let’s open a terminal and repeat the same traceroute exercise we did before, now considering pods on different nodes.

## Practical example:

1.  Create one pod named httpd, use the **httpd** image, and be sure it is placed on node-1:

\[vagrant@controlplane ~\]$ kubectl run httpd --image=httpd \\  
 --overrides='{"spec": { "nodeSelector": {"kubernetes.io/hostname": "node-1"}}}'

2\. Check the pod’s status to get the IP address (e.g., 10.244.1.3) of pod httpd. This pod is the target:

\[vagrant@controlplane ~\]$ kubectl get pods -o wide

3\. Create one pod name traceroute, use the **busybox** image, and run a **traceroute** command with the IP address of httpd pod as an argument. Be sure this pod is placed on node-2:

\[vagrant@controlplane ~\]$ kubectl run traceroute --image=busybox \\  
 --overrides='{"spec": { "nodeSelector": {"kubernetes.io/hostname": "node-2"}}}' \\  
 -- traceroute 10.244.1.3

4\. Check the logs output in the traceroute pod:

\[vagrant@controlplane ~\]$ kubectl logs traceroute  
traceroute to 10.244.1.3 (10.244.1.3), 30 hops max, 46 byte packets  
 1  10.244.2.1 (10.244.2.1)  0.007 ms  0.008 ms  0.004 ms  
 2  192.168.56.11 (192.168.56.11)  0.575 ms  0.411 ms  0.396 ms  
 3  10.244.1.3 (10.244.1.3)  0.614 ms  0.580 ms  0.498 ms

The traceroute output shows *three hops* this time:

-   The first is 10.244.2.1, the gateway address configured in the *cni0* network interface at node-2.
-   The second hop is 192.168.56.11, the IP address the eth1 network interface has for node-1.
-   The third and last hop is 10.244.1.3, the IP address attached to the *cni0* network of node-1 and assigned to the pod we wanted to test connectivity for.

We can conclude that: *when pods are in different nodes, the communication is routed between the cni0 and eth1 interfaces attached to each node.*

In this article, we have delved into Kubernetes networking within the context of VirtualBox, providing command-line examples and illustrations that shed light on pod-to-pod communication. However, it’s important to recognize that our exploration has only scratched the surface of this vast topic.

As you continue your exploration of Kubernetes networking, it becomes evident that understanding how network packages flow is essential for effectively identifying and troubleshooting networking issues. Whether building your own Kubernetes cluster learning lab or managing an existing deployment, this knowledge is invaluable when unexpected communication issues arise.

I encourage you to let your curiosity guide you as you continue learning and exploring the intricacies of Kubernetes networking. Trace pod-to-pod traffic, uncover the underlying mechanisms, and embrace the challenge of maintaining reliable and efficient communication within your Kubernetes environment. Remember, there is always more to discover, so keep expanding your expertise and enhancing your ability to troubleshoot and optimize Kubernetes networking.

-   [The Kubernetes Networking Guide](https://www.tkng.io/)
-   [The Kubernetes Networking Guide — Flannel](https://www.tkng.io/cni/flannel/)
-   [Flannel Project](https://github.com/flannel-io/flannel)
-   [OSI Model — Data Link Layer](https://osi-model.com/data-link-layer/)
-   [A visual guide to Kubernetes networking fundamentals](https://opensource.com/article/22/6/kubernetes-networking-fundamentals)
-   [Kubernetes.io — Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
-   [How a Kubernetes Pod Gets an IP Address](https://ronaknathani.com/blog/2020/08/how-a-kubernetes-pod-gets-an-ip-address/)