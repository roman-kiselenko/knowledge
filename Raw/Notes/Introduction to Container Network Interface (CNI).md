---
title: Introduction to Container Network Interface (CNI)
source: https://addozhang.medium.com/introduction-to-container-network-interface-cni-25309a64b23e
clipped: 2023-09-05
published: 
category: network
tags:
  - k8s
  - network
  - cni
read: false
---

This is the second note on Kubernetes network learning.

-   [Deep Dive into Kubernetes Network Model and Communication](https://medium.com/@addozhang/deep-dive-into-kubernetes-network-model-and-communication-57a2bffc852e)
-   Introduction to Container Network Interface (CNI) (this article)
-   Source Code Analysis: Using CNI from kubelet and Container Runtime
-   Learning Kubernetes VXLAN Network with Flannel
-   Kubernetes network learning with Cilium and eBPF

In the article [Deep Dive into Kubernetes Network Model and Communication](https://medium.com/@addozhang/deep-dive-into-kubernetes-network-model-and-communication-57a2bffc852e), we explored how network namespaces work in the Kubernetes network model and analyzed the traffic transmission path between pods through examples. The entire transmission process requires the participation of various components, which have the same lifecycle as pods and are created and destroyed along with them. The maintenance of containers is delegated by the kubelet to the container runtime, and the container’s network namespace is jointly completed by the container runtime and network plugins.

-   Creating the network namespace for pods (containers)
-   Creating interfaces
-   Creating veth pairs
-   Configuring the namespace network
-   Setting up static routes
-   Configuring Ethernet bridging
-   Allocating IP addresses
-   Creating NAT rules
-   …

In the previous article, we also mentioned that different network plugins have different implementations for the Kubernetes network model, mainly focusing on the communication between pods across nodes. Users can choose the appropriate network plugin based on their needs, and this cannot be done without the Container Network Interface (CNI). These network plugins implement the CNI standard and integrate well with container orchestration systems and runtimes.

CNI is a project under CNCF that provides essential [specification](https://github.com/containernetworking/cni/blob/main/SPEC.md), a [library](https://github.com/containernetworking/cni/blob/main/libcni) for CNI integration with applications, a CLI tool called `[cnitool](https://github.com/containernetworking/cni/blob/main/cnitool)` for CNI plugins, and [reference plugins](https://github.com/containernetworking/plugins). The latest version at the time of this article's publication is v1.1.2.

CNI focuses on container network connectivity and cleaning/releases of allocated resources upon container destruction. Due to this focus, CNI remains simple and is widely [supported](https://github.com/containernetworking/cni/blob/main/README.md#who-is-using-cni) despite the rapid development of containers.

The CNI specification covers the following parts:

-   Network configuration file format
-   Protocol for interaction between container runtime and network plugins
-   Execution flow of plugins
-   Execution flow when delegating to other plugins
-   Data types of execution results returned to the runtime

Here is an example of the configuration from the specification. The [specification](https://github.com/containernetworking/cni/blob/main/SPEC.md#section-1-network-configuration-format) defines the format of network configuration, including required fields, optional fields, and the functionality of each field. The example defines a network named `dbnet` and configures two plugins: `bridge` and `tuning`.

CNI plugins are generally divided into two types:

-   Interface plugins: Used to create network interfaces, such as \`bridge

\` in the example.

-   Chained plugins: Used to adjust existing network interfaces, such as `tuning` in the example.

```yaml
{  
  "cniVersion": "1.0.0",  
  "name": "dbnet",  
  "plugins": \[  
    {  
      "type": "bridge",  
        
      "bridge": "cni0",  
      "keyA": \["some more", "plugin specific", "configuration"\],

      "ipam": {  
        "type": "host-local",  
        // ipam specific  
        "subnet": "10.1.0.0/16",  
        "gateway": "10.1.0.1",  
        "routes": \[  
            {"dst": "0.0.0.0/0"}  
        \]  
      },  
      "dns": {  
        "nameservers": \[ "10.1.0.1" \]  
      }  
    },  
    {  
      "type": "tuning",  
      "capabilities": {  
        "mac": true  
      },  
      "sysctl": {  
        "net.core.somaxconn": "500"  
      }  
    },  
    {  
        "type": "portmap",  
        "capabilities": {"portMappings": true}  
    }  
  \]  
}
```

CNI provides four different operations for container runtime as mentioned in the [specification](https://github.com/containernetworking/cni/blob/master/SPEC.md#cni-operations):

-   **ADD**: Adds a container to the network or modifies the configuration.
-   **DEL**: Removes the container from the network or cancels the modification.
-   **CHECK**: Checks if the container network is functioning properly and returns an error if there are any issues.
-   **VERSION**: Displays the version of the plugin.

The specification defines the input and output content for these operations. The key fields include:

-   `CNI_COMMAND`: One of the four operations mentioned above.
-   `CNI_CONTAINERID`: Container ID.
-   `CNI_NETNS`: Isolation domain of the container. If using network namespaces, this value is the address of the network namespace.
-   `CNI_IFNAME`: Interface name to be created inside the container, such as `eth0`.
-   `CNI_ARGS`: Additional parameters passed during execution.
-   `CNI_PATH`: Path to the plugin executable file.

CNI refers to the network configuration operations of `ADD`, `DELETE`, and `CHECK` as attachments.

The operations for container network configuration require the collaboration of one or more plugins, so the plugins have a certain execution order. For example, in the previous example configuration, the interface needs to be created first before it can be fine-tuned.

Taking the `ADD` operation as an example, the general execution order is to first execute the `interface plugin` and then execute the `chained plugin`. The **output** `PrevResult` of the previous plugin and the configuration of the next plugin are used as the **input** for the next plugin. If it is the first plugin, the network configuration will be part of the input. The plugin can use the `PrevResult` from the previous plugin as its output or combine it with its own operations to update the `PrevResult`. The output `PrevResult` of the last plugin is returned as the execution result of CNI to the container runtime, and **the container runtime will save and use this result as the input for other operations**.

The execution order for `DELETE` is the opposite of `ADD`, where the configurations on the interface are removed or the allocated IP addresses are released before deleting the container network interface. The

input for the `DELETE` operation is the result of the `ADD` operation saved by the container runtime.

In addition to defining the execution order of plugins in a single operation, [CNI also provides specifications](https://github.com/containernetworking/cni/blob/main/SPEC.md#lifecycle--ordering) for parallel operations and repeated operations.

There are some operations that, for various reasons, cannot be reasonably implemented as loosely linked plugins. Instead, CNI plugins may want to delegate certain functions to other plugins. A common example is IP Address Management (IPAM), which mainly involves allocating/reclaiming IP addresses for container interfaces and managing routes.

CNI defines a third type of plugin, the IPAM plugin. CNI plugins can call the IPAM plugin at the appropriate time, and the IPAM plugin returns the execution result to the delegating plugin. The IPAM plugin performs operations based on specified protocols (such as DHCP), data in local files, or information in the `ipam` field of the network configuration file: IP allocation, gateway settings, route settings, etc.

```json
"ipam": {  
  "type": "host-local",  
    
  "subnet": "10.1.0.0/16",  
  "gateway": "10.1.0.1",  
  "routes": \[  
      {"dst": "0.0.0.0/0"}  
  \]  
}
```

Plugins can return one of the following three types of results, and the specification defines the [format of the results](https://github.com/containernetworking/cni/blob/main/SPEC.md#section-5-result-types):

-   Success: Includes the `PrevResult` information, such as returning the `PrevResult` after the `ADD` operation to the container runtime.
-   Error: Includes necessary error information.
-   Version: This is the result of the `VERSION` operation.

The CNI library refers to `[libcni](https://github.com/containernetworking/cni/tree/main/libcni)`, which is used for CNI integration with applications and defines interfaces and configurations related to CNI.

```go
type CNI interface {    
   AddNetworkList(ctx context.Context, net \*NetworkConfigList, rt \*RuntimeConf) (types.Result, error)    
   CheckNetworkList(ctx context.Context, net \*NetworkConfigList, rt \*RuntimeConf) error    
   DelNetworkList(ctx context.Context, net \*NetworkConfigList, rt \*RuntimeConf) error    
   GetNetworkListCachedResult(net \*NetworkConfigList, rt \*RuntimeConf) (types.Result, error)    
   GetNetworkListCachedConfig(net \*NetworkConfigList, rt \*RuntimeConf) (\[\]byte, \*RuntimeConf, error)  

   AddNetwork(ctx context.Context, net \*NetworkConfig, rt \*RuntimeConf) (types.Result, error)    
   CheckNetwork(ctx context.Context, net \*NetworkConfig, rt \*RuntimeConf) error    
   DelNetwork(ctx context.Context, net \*NetworkConfig, rt \*RuntimeConf) error    
   GetNetworkCachedResult(net \*NetworkConfig, rt \*RuntimeConf) (types.Result, error)    
   GetNetworkCachedConfig(net \*NetworkConfig, rt \*RuntimeConf) (\[\]byte, \*RuntimeConf, error)   ValidateNetworkList(ctx context.Context, net \*NetworkConfigList) (\[\]string, error)    
   ValidateNetwork(ctx context.Context, net \*NetworkConfig) (\[\]string, error)    
}
```

Taking the part of adding a network as an example:

```go
func (c \*CNIConfig

) addNetwork(ctx context.Context, name, cniVersion string, net \*NetworkConfig, prevResult types.Result, rt \*RuntimeConf) (types.Result, error) {    
   ...  
   return invoke.ExecPluginWithResult(ctx, pluginPath, newConf.Bytes, c.args("ADD", rt), c.exec)    
}
```

The logic can be summarized as follows:

1.  Find the executable file.
2.  Load the network configuration.
3.  Execute the `ADD` operation.
4.  Handle the result.

In this article, we have learned about the CNI specification, the execution flow of network plugins, and gained a general understanding of the abstracted network management interface of CNI.

In the next article, we will analyze the source code to understand how the kubelet, container runtime, and CNI network plugins interact with each other.

-   [https://www.tigera.io/learn/guides/kubernetes-networking/kubernetes-cni/](https://www.tigera.io/learn/guides/kubernetes-networking/kubernetes-cni/)
-   [https://github.com/containernetworking/cni](https://github.com/containernetworking/cni)
-   [https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)