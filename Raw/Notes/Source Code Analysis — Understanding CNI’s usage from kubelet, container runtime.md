---
title: Source Code Analysis — Understanding CNI’s usage from kubelet, container runtime
source: https://addozhang.medium.com/source-code-analysis-understanding-cnis-usage-from-kubelet-container-runtime-24d72f29466b
clipped: 2024-02-05
published: 
category: network
tags:
  - k8s
  - container-runtime
read: false
---

This is the third note on learning Kubernetes networking.

-   [Deep Dive into Kubernetes Network Model and Communication](https://medium.com/@addozhang/deep-dive-into-kubernetes-network-model-and-communication-57a2bffc852e)
-   [Understanding the Container Network Interface (CNI)](https://addozhang.medium.com/introduction-to-container-network-interface-cni-25309a64b23e)
-   [Source code analysis: how kubelet and container runtime work with CNI](https://medium.com/@addozhang/source-code-analysis-understanding-cnis-usage-from-kubelet-container-runtime-24d72f29466b)
-   [Learning Kubernetes VXLAN network from Flannel](https://addozhang.medium.com/learning-kubernetes-vxlan-network-with-flannel-2d6a58c95300)
-   [Kubernetes network learning with Cilium and eBPF](https://addozhang.medium.com/kubernetes-network-learning-with-cilium-and-ebpf-aafbf3163840)
-   …

In the previous article, through the interpretation of the CNI specification, we understood the operations and processes of network configuration. Among the several operations in the network, besides `CNI_COMMAND`, there are three other parameters that are almost always provided: `CNI_CONTAINERID`, `CNI_IFNAME`, and `CNI_NETNS`. These parameters all come from the container runtime. This article will analyze the use of CNI by combining the source code of Kubernetes and [Containerd](https://containerd.io/).

The source code of Kubernetes is from the branch `release-1.24`, and that of Containerd is from the branch `release/1.6`.

!\[\[runtime-with-cni.png\]\]

In the previous [kubelet source code analysis](https://mp.weixin.qq.com/s/O7k3MlgyonNtOUxNPrN8lg), it was mentioned that `Kubelet#syncLoop()` continuously monitors changes from **files**, **apiserver**, **http** to update the status of the pod. When writing that article, the analysis ended here. Because the work after this is handed over to the container runtime to complete the creation and running of *sandbox* and various containers, see `kubeGenericRuntimeManager#SyncPod()`.

`kubelet` encapsulates requests for creating and running sandbox and containers, calls the container runtime interface, and delegates the specific work to the container runtime (Container Runtime Interface, abbreviated as CRI, will be studied in the future).

## Reference Source Code

-   `[pkg/kubelet/kubelet.go:1985](https://github.com/kubernetes/kubernetes/blob/release-1.24/pkg/kubelet/kubelet.go#L1985)`
-   `[pkg/kubelet/kuberuntime/kuberuntime_manager.go:711](https://github.com/kubernetes/kubernetes/blob/release-1.24/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L711)`

Remember in the [first article of the series](https://atbug.com/deep-dive-k8s-network-mode-and-communication/), when we viewed namespaces on the node, the process of the network namespace was `/pause`.

lsns -t net  
        NS TYPE NPROCS    PID USER     NETNSID NSFS                                                COMMAND  
4026531992 net     126      1 root  unassigned                                                     /lib/systemd/systemd --system --deserialize 31  
4026532247 net       1  83224 uuidd unassigned                                                     /usr/sbin/uuidd --socket-activation  
4026532317 net       4 129820 65535          0 /run/netns/cni-607c5530-b6d8-ba57-420e-a467d7b10c56 /pause

When Kubernetes creates a pod, it first creates a sandbox container (using the `pause` image, execute `/pause` to enter sleep mode when starting). We know that Kubernetes allows multiple containers in a pod, and this sandbox container creates and maintains the network namespace, and other containers in the pod will join this namespace. Because the pause image is simple enough, it will not cause errors that cause the network management space to be deleted when an error occurs. [The sandbox container plays a crucial role](https://www.ianlewis.org/en/almighty-pause-container), it serves as the process with PID 1 in the process tree of the PID process space, and other container processes take it as the parent process. When other container processes become orphan processes, they can be cleaned up.

CRI’s `RuntimeServiceServer` defines the service interface provided by the runtime to the outside. In addition to managing sandbox and container-related operations, there are also streaming-related operations, namely common `exec`, `attach`, `portforward`. For streaming related content, you can refer to a previous article [《Source Code Analysis of the Working Principle of kubectl port-forward》](https://mp.weixin.qq.com/s/oDSTJBUh1B-6mf7DX7g4Zg).

Let’s look at the container-related part.

Containerd’s `criService` implements the `RuntimeServiceServer` interface. The request to create a sandbox container enters the `criService` processing flow through the CRI [UDS (Unix domain socket)](https://en.wikipedia.org/wiki/Unix_domain_socket) interface `/runtime.v1.RuntimeService/RunPodSandbox`. In `criService#RunPodSandbox()`, it is responsible for creating and running sandbox containers and ensuring that the container status is normal.

1.  The container runtime first initializes the container object and generates the necessary parameters `CNI_CONTAINERID`
2.  Then it will create the pod network namespace, generate the necessary parameters `CNI_NETNS`
3.  Then call the CNI interface to configure the network space of the pod, such as creating a network interface, allocating IP addresses, creating veth, setting routes, and a series of operations. These operations are implemented by the specific network plugin. There are differences in implementation between different plugins. After understanding the specifications, the network configuration is not difficult. Among them, 2 and 3 may be executed multiple times:
4.  Read network configuration
5.  Find binary files
6.  Execute binary files
7.  Feedback results to the container runtime
8.  Finally, it is to create the sandbox container. This process is related to the type of operating system and will call the corresponding operating system method to complete the creation of the container.

If you are learning containers from scratch, I recommend reading Ivan Velichko’s [《Learning Containers From The Bottom Up》](https://iximiuz.com/en/posts/container-learning-path/)

## Reference Source Code:

-   [pkg/cri/server/sandbox_run.go:61](https://github.com/containerd/containerd/blob/release/1.6/pkg/cri/server/sandbox_run.go#L61)
-   [pkg/cri/server/sandbox_run.go:422](https://github.com/containerd/containerd/blob/release/1.6/pkg/cri/server/sandbox_run.go#L422)

Next, it is to create other containers in the pod: temporary (`ephemeral`), initialization (`init`), and ordinary containers. When creating these containers, the container will be added to the network namespace of the sandox. This is not expanded here, the detailed logic can refer to containerd's `containerStore#Create()`.

## Reference Source Code

-   kubernetes: `[pkg/kubelet/kuberuntime/kuberuntime_manager.go:913](https://github.com/kubernetes/kubernetes/blob/release-1.24/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L913)`
-   containerd: `[pkg/cri/server/container_create.go:51](https://github.com/containerd/containerd/blob/release/1.6/pkg/cri/server/container_create.go#L51)`

Following the introduction of CNI specifications in the last article, this time I introduced the use of CNI, as well as the interaction with the container runtime and the creation process of Pod.

Different CNI plugins implement different network functions. In the next article, I will take [Flannel](https://github.com/flannel-io/flannel) as an example to understand the implementation of CNI and Kubernetes VXLAN network.

Why introduce flannel? Because one of the development environments I often use, [k3s](https://k3s.io/), uses flannel network by default. Another development environment is [k8e](https://getk8e.com/), k8e uses [Cilium](https://cilium.io/) by default, and cilium’s cni is also one of the series of articles.