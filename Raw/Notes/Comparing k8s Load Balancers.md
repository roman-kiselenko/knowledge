---
title: Comparing k8s Load Balancers
source: https://medium.com/thermokline/comparing-k8s-load-balancers-2f5c76ea8f31
clipped: 2024-09-19
published: 
category: k8s
tags:
  - networking
  - cni
read: false
---

In this article we discuss three open source load-balancer controllers that can be used with any distribution of Kubernetes.

-   [MetalLB.](https://metallb.universe.tf/) The popular and most well know load-balancer controller
-   [PureLB.](https://purelb.gitlab.io/docs/) The newest addition. (Full disclosure, I am involved in the development of PureLB)
-   [OpenELB](https://github.com/kubesphere/openelb). A relatively recent addition, initially focused on routing only

Adding a controller that implements Service Type LoadBalancer functionality is a key networking component necessary for simple, scaleable cluster operation.

-   Enable controlled external access to cluster services/applications
-   External resources are pre-configured
-   Easy to integrate with automated workflows (CI/CD)

The first is obvious however the second two points are just as critical in designing a load-balancer solution for your cluster. Depending on the deployment model, it’s common that the team responsible for ensuring reliable networking to the cluster is different from the team running the cluster(s). Pre-configuration allows the network team to help get things setup and leave operation to the cluster team. This facilitates easy integration with CI/CD because the use of load-balancer resources is now part of the standard k8s “application” definition and workflow.

These independent load balancers can be run in any k8s environment unlike the integrated load-balancers used in Public Clouds or implementation specific bundled solutions such as Klipper in K3s.

All load-balancer controllers expose services, how each of them achieve this varies, and this difference impacts operational behaviors and failure modes. Each of these load-balancer controllers includes an address allocator and one or more mechanisms to add addresses to the network, the key difference is the techniques used to add and manage addresses.

## Allocating addresses

All of the load-balancers include integrated IP address Management (IPAM) used to allocate IP address from pools to services. They all have similar operation: they watch the Services API for type LoadBalancer without an IP address and update kube-api with an IP address. In turn, other load-balancer components and kube-proxy are watching the Services API for events of type LoadBalancer containing an IP address and use that information to trigger addition of addresses to the network. There is one common problem that impacts allocator and the other components operation. The type used for the IP address in the Services API is a string containing an address in the form of a.b.c.d (or a::z for IPv6). An IP address used in network is often referred to as a prefix because it includes the subnet mask, e.g. a.b.c.d/mask (a::z/mask). In golang, the type used would be an ipNet, however kube-api uses a string for this field. This is an important detail as it must be handled by the process that adds the addresses.

Generally there are two primary types of operation.

## Add addresses to local interfaces

Adding address to a local interface is the first requirement in a small cluster, but can also be useful in larger clusters to advertise services that require local access from non-k8s infrastructure components. However, adding a local address is not as simple as it may first appear. An address on a local network interface must be unique, however k8s orchestrates all of it nodes in a uniform manner. Therefore a mechanism must be used to identify and ensure that an address is only added to a single node, hopefully on the desired network. This is further complicated if kube-proxy, the mechanism used to program distribution within the cluster, is configured in IPVS mode.

> IPVS. The IPVS implementation in kube-proxy increases scalability by reducing the use of iptables. Instead of using PREROUTING in the iptables input chain, a dummy interface is created called kube-ipvs0. Kube-proxy adds load balanced addresses to kube-ipvs0. By default the Linux kernel will answer ARP/ND requests from any interface for addresses on any interface. The reason for this behavior is to increase the probability of successful connection. However this creates problems when the addresses are allocated from the local subnet as all hosts respond to ARP/ND resulting in duplicate ARP messages. All of the discussed Load Balancers require setting the host ARP/ND mode to “strict”, good practice on servers with multiple interfaces.

Solving operational problems requires an understanding of how addresses are added and which node they are added. Its important to know how k8s permits the addition of addresses to a node. A POD that uses the Node’s network interfaces is set to use hostNetwork:true with allowedCapabilities NET\_ADMIN and in some cases NET\_RAW set. Its easy to figure out which PODS are set to use the host network, the POD address will be the nodes address.

## Add addresses to routers for distribution

Adding allocated addresses to the network routing tables is a more scalable and redundant solution. Routing allows the same address to be advertised by multiple k8s nodes. This allows the load-balancers to populate routing tables in switches to use Equal Cost Multipath providing a layer of packet distribution before traffic enters the cluster. Multiple paths are advertised making this a more redundant solution. However its important to remember that the Service load-balancer needs to cooperate with kube-proxy, the in-cluster traffic distribution mechanism.

## Service Traffic Policy

externalTrafficPolicy is an important configuration setting. It provides two choices on how traffic will be distributed to the cluster and within the cluster. When set to Cluster (default), each node is configured by kube-proxy to receive and distribute traffic equally throughout the cluster. When set to local, kube-proxy expects to receive traffic only on nodes where PODs are running, The local setting is widely used for external applications because this mode removes the need for kube-proxy to NAT resulting in the origin source IP address being present, instead of a “natted” node address, at the POD. The load-balancer must accommodate these behaviors for reliable traffic delivery.

## Open Source

All of these load-balancer controllers are open source, therefore documentation coverage also varies. In many cases reading source code is necessary to understand the detailed operations.

The first and widely deployed LoadBalancer started life as a 10% project at Google and until recently was the only choice for teams seeking an open source software only solution to Load Balancing. It operates in two distinct modes described as L2 mode and BGP mode.

MetalLB is configured using a configmap.

The controller consists of two components.

-   controller. Allocates IP addresses. One per cluster
-   speaker. Configures node networking. Runs on all nodes providing access to IP addresses

## Allocator (controller)

The MetalLB allocator is quite straightforward. Pools of addresses are configured by the configmap, however the configuration of its modes are also part of the address pool configuration.

The controller behavior is consistent, the speaker implements to two modes of operation. In MetalLB these modes are configured as “protocols” in the configmap pools.

## **L2 Mode**

There are two key components working together in MetalLB to add the address to a single node. The first is node election using memberlist. This is a simple election scheme that uses the service name to ensure that each node selects the same winner, a single node where the allocated ip address will be answered.

I use the term answered, because in “L2” the speaker process implements ARP and ND. Kube-proxy has added the necessary configuration to forward traffic to the target POD, an ARP/ND response is still necessary to initiate communications.

In L2 mode MetalLB does not use Linux networking to answer ARP/ND, processing is implemented in the speaker POD executable. Therefore its ARP/ND processing is not visible to Linux and its networking tools. Identifying which node is answering ARP/ND requests requires either the use of APR/ND tools on other network hosts or inspecting the POD speaker logs.

Metallb is not aware of the address ranges used by the Nodes, the ARP/ND process answers any network address configured in the configmap. This can result in addresses being allocated in L2 mode that are not reachable. Care especially needs to be taken when the node network spans multiple host subnets, there is not guarantee that the added address will be located on a node with local connectivity.

## **BGP Mode**

MetalLB also operates in BGP mode. BGP provides a mechanism to advertise prefixes (addr/mask) to neighboring routers. Mechanisms using routing leverage the layer 3 forwarding of Linux combined with a routing protocol to provide destination information.

The configmap is configured with the BGP autonomous system information, peer information, and pool, identified by protocol bgp.

The speaker, running as a daemonset on each node, establishes a BGP peer connection from every node to configured peer router. AS numbers identifies the peer as External or Internal.

When an address is allocated from the pool, each speaker advertises that address as a host route. The controller simply takes the allocated string from kube-api and turns it into an ipnet by adding a /32 prefix making all allocated addresses host routes. This is the address that is advertised to BGP neighbors. This can be problematic as some routers that will not accept /32 routes from BGP. MetalLB has some BGP address aggregation functionality, however that does not change the /32 advertisements, it simply tells the upstream router to aggregate and the peer router will still be populated with /32 routes.

MetalLB has its own BGP implementation. This is important for three reasons.

1\. MetalLB uses the nodes IP address and is a router. This means that another process running in the default network namespace, such as a software router cannot use that host address to peer to the same router. In larger networks, this can result in very complex BGP configurations.

2\. The BGP functionality has no way of integrating with the Linux routing tables and can only be used for advertising prefixes created by MetalLB.

2\. The BGP functionality is limited by the level of support in MetalLB, and features available in other software routers need to be specifically developed for MetalLB. MetalLB has some additional BGP features such as aggregation and community support but does not have the features considered necessary in standard routers.

Both modes can be used concurrently, each requires specific configuration.

## Traffic Policy.

MetalLB supports this setting, however setting externalTrafficPolicy to local in L2 mode will likely result in packet loss and should not be used. In BGP mode only the nodes with PODs are advertised. Traffic will be equally distributed among nodes where PODs are running, however each node will receive an equal share irrespective of the number of pods on that node.

## Configuration

MetalLB is configured using configmaps. When originally developed this was the primary mechanism used for configuration and is still used by many controllers today. The problem with configmaps is that there is no way to verify the configuration prior to it being read by the application POD. Therefore a incorrect configuration results in POD failure or incorrect operation and can only be debugged “post failure” by reviewing the individual POD logs.

Further adding to the configmap complexity, MetalLB has an odd configuration behavior worth knowing before its deployed. Incorrect configuration changes cause the configmap to be marked as stale. The old configuration, stored as an annotation in the configmap, continues to be used. For example, once defined and in use, changing address pools is difficult. The configuration of pools cannot be changed if a service uses an address from an existing pool. Changing the pool marks the configuration stale and MetalLB continues to use the same address pool, new services are allocated from the old pool. The only way to know that the configuration is stale is to check the MetalLB POD logs. Should speakers be restarted manually or through node failures, the old address allocations will remain, however if the controller is restarted all services will be renumbered to the new configuration. If the controller or speaker is restarted with an invalid configuration, and the configuration is invalid, the last configuration is loaded from the configmap annotation. Be careful to ensure that every configmap change is valid by checking a speaker log. A “stale” config with a lot of addresses allocated can be difficult to back out without suffering downtime.

MetalLB’s documentation is sufficient but sparse. To understand how it operates, it is necessary to read the source code.

## IPv6

Upon source code inspection, you will notice that some of MetalLB’s components support IPv6, however as a system, MetalLB does not support IPv6. Adding an address pool will cause the configuration loaded in the controller and the speaker to fail and become “stale”.

## MetalLB Summary.

MetalLB does not use native Linux Networking making it difficult to troubleshoot using standard Linux tools. It has its own, unique BGP implementation that lacks features used to manage larger scale BGP infrastructures. Adding them requires BGP protocol development, while possible it would take a lot of development to make the BGP implementation fully featured. Because it is a BGP router, a node using BGP for CNI cannot concurrently peer the BGP for the CNI and BGP from MetalLB to the same upstream router. There are topological workarounds, but they add significant complexity to the routing infrastructure.

PureLB is the newest open source load-balancer. Inspecting the code will reveal that it started life as a fork of MetalLB however operates very differently. PureLB does not have “operating” or protocol modes, it automatically allocates addressed from Service Groups containing address pools to different interfaces. Those interfaces respond to local network requests or distribute routes via independent routing software.

PureLB is configured using Custom Resources.

The controller consists of two components.

-   allocator. Allocates IP addresses. One per cluster
-   lbnodeagent . Configures node networking. Runs on all nodes providing access to IP addresses

PureLB does not have “protocol modes” simply Service Groups containing addresses. PureLB uses Linux Network functionality to add addresses that are used locally and redistributed by Routers. There is minimal configuration necessary, however interface defaults can be overridden in a Custom Resource where specific network interfaces are desired.

## Allocator (allocator)

The allocator in PureLB includes an integrated IPAM, but also supports external IPAM systems. Currently the only external IPAM supported is the popular open source NetBox. The allocator is configured using Custom Resources. Each address pool is contained in a unique Service Group. The Service Group contains other network information to allow the lbnodeagent to construct the desired prefix or IPNet that is added. The allocator behavior is consistent irrespective of how the address is used and no “protocol” configuration is required.

## Local Network Addresses

In PureLB a “local” address is an address that is added as a secondary address to the nodes primary network interface. PureLB identifies the primary network adapter by identifying which interface has the default route. When an address is allocated from an address pool where that address matches the primary network interface prefix, PureLB elects a node using memberlist and adds that address to the interface as a secondary address.

As the address is added as a secondary to a network interface, Linux networking responds to ARP/ND requests, PureLB does not implement its own ARP/ND process. Further the address is visible using standard Linux networking tools, such as iproute2, so finding where an address is allocated can be undertaken with standard Linux tools.

In addition when PureLB allocates an address, the service is annotated with the IPAM source and the allocated address. In the case of a local address, the elected node and interface is also added to the service as annotations.

PureLB is aware of the node networks, therefore only Service Group Pools matching local networks will be applied to local interfaces. This ensures that addresses that have local connectivity established will be accessible and addresses that require routing will use a the virtual interface mechanism.

## Virtual Network Addresses

PureLB adds addresses from Service Group pools that do not match a local node interface to a virtual interface called kube-lb0. The lbnodeagent adds the interface at POD startup.

The virtual interface can be used to add any network that will be accessible via routing, To enable efficient routing PureLB allows the address to be added with a default or configured address aggregation mask. This mask, added in the Service Group, is applied to create the ipNet that is added to the virtual interface. The Linux kernel takes care of adding the address to the interface and creating the correct routes. This allows a subnet to be advertised once in cases where externalTrafficPolicy: cluster is used, or a host-route to be advertised when it’s set to local. Network designers use address aggregation to control the size of routing tables, supporting aggregation provides large scale network flexibility.

## Routing Protocols

PureLB does not directly implement any routing protocols requiring the addition of a software router to distribute distribute reachability for addresses added to the virtual interface. Where the cluster does not have routing software in use for infrastructure or CNI, PureLB provides a bundled, pre-configured BIRD router daemonset. The router is configured to use a common function called “redistribution” to read the routes associated with the virtual interface and distribute them using the configured routing protocols.

Using this mechanism, any routing protocols supported by the routing software used can be used to distribute load-balancer addresses and the development of routing protocols is decoupled from the load-balancer. The packaged BIRD router supports BGP, OSPF, RIP for both IPv4 and IPv6. Others such as FRR support ISIS and IGRP as well, commercial routing software can also be used. One of the key benefits of this mechanism is that the routing software provides a complete implementation each routing protocol allowing maximum flexibility in designing the network and how routes are distributed. In a network that uses OSPF for reachability, load-balancer address can be distributed using OSPF instead of requiring the addition of BGP.

Routing protocols are needed when the CNI is not configured to use tunneling and the POD network spans multiple subnets. In some cases the CNI’s include routing functionality as part of their operation. In these cases, PureLB can avoid the issues caused by attempting to run two routing instances on the same nodes. The CNI routing process simply redistributes routes from the virtual interface and applies any necessary routing policies creating a simpler, more natural routing topology.

## Traffic Policy

PureLB supports externalTrafficPolicy:local for Virtual Network Addresses. A virtual network address is only added to a node when a POD is present. Traffic will be equally distributed among nodes where PODs are running, however each node will receive an equal share irrespective of the number of PODs on that node. PureLB does not support externalTrafficPolicy:local for Local Network Addresses, in fact if you attempt to set externalTrafficPolicy to local for a local network address, PureLB will reset it to Cluster to ensure that kube-proxy is correctly configured and traffic will reach all of the selected PODs

## Configuration

Configuration of PureLB is via Custom Resources. There is a CR for the overall system configuration and a CR for each Service Group. As each CR is independent and CRs can be validated at creation time, PureLB maintains a consistent configuration and address pools can be changed as necessary. PureLB will never change an address that has been allocated, it expects address changes are a deliberate behavior therefore an address change requires the add/delete of a service.

As PureLB operates it adds annotations to the services where it provides load-balancer services as well as updating the service event logs, therefore the key service information and status can be extracted from the service without requiring the need to inspect POD logs.

## IPv6

PureLB includes support for IPv6. A Service Group is defined that includes the address and network information and when that pool is used, IPv6 addresses are added, either locally or virtually. To distribute via routing protocols, the appropriate IPv6 routing protocols need to be configured in a software router, the packaged BIRD router supports IPv6.

## Summary

The new kid on the block differentiates itself by providing external IPAM, IPv6 and more importantly leverages Linux networking. Instead of attempting to reinvent network and routing protocols, PureLB enables the use of all of Linux and Routing capabilities. It is easier to implement PureLB with complex infrastructures where routing is required for other aspects of reachability, such as CNI, Storage or management networks and is already present in the cluster. Inspecting the code shows how different PureLB is from the version of MetalLB from which it was forked and its simplicity is demonstrated by the fact that it has about half the lines of code.

Porter is a relatively recent addition to open source load-balancers. It is an independent development. Originally BGP only, recently local address support has been added. Porter differs in the way it distributes functionality between the controller and agents. Porter also implements BGP but requires a BGP router on the local network with a custom port configuration.

Porter is configured using Custom Resources.

The controller consists of two components:

-   porter-manager. Allocates addresses and configures network functionality.
-   porter-agent. Gather node specific information used by porter manager to configure the network.

## Allocator

The allocator is part of the porter-manager. There is a single instance of the Porter Manager. Pools are configured in as customs resource called an EIP. This contains the address and the protocol used for address reachability.

Note that porter does not have any concept of a default load-balancer address pool, an annotation is always required.

Porter centralizes network configuration in the porter-manager, with the porter-agent simply collecting events, adding node specific information and returning that information to the porter-manager.

## L2

In L2 mode Porter does not use the Linux networking. The porter-manager executable implements its own ARP process. When an address is allocated from the EIP pool, the porter-manager answers requests for that address on behalf of a target node. The ARP process for services is centralized in the manager. While this mechanism should work, it is not strictly in compliance with how ARP processing should operate. The porter-manager answers initial requests because they are broadcasts. However APR cache processing uses unicast ARP messages to verify cache entries, falling back to broadcast only when unicast ARP fails. The porter-manager will not answer unicast ARP requests. This increases the volume of broadcast traffic and depends upon a non-standard behavior for operation. Layer2 switch security mechanisms that track hardware mac addresses will likely interfere with this operation causing the L2 functionality to fail.

Compounding the problems with the L2 implementation, when a deployment is scaled or a daemonset is present, porter-manager incorrectly responds with the mac address of every node where a pod is running, effectively behaving as if there are duplicate IPaddresses on the network. The only time L2 mode can be used is for a single POD on the cluster.

Finally, continued operation depends on the porter-manager, the porter-manager uses standard k8s POD failure detection mechanisms, therefore the porter-manager can take a significant amount of time to restart, impacting new and existing services as their will be no ARP responses.

## BGP

Porter implements BGP using the gobgp, however it implements a very small subset of gobgp functionality. The Porter BGP implementation has only the necessary functionality to peer with a router on the local network segment.

Porter establishes as single BGP peer for the cluster from porter-manager node. The BGP peer must be on the local segment as it is not possible to configure BGP multi-hop.

When a service is defined, the porter-manager advertises the allocated address with a next hop of the node where the container is running. Scaling a deployment so the PODs are distributed over multiple nodes results in additional routes being added for ECMP. Each allocated address is converted to a host route (/32). However this traffic policy behavior is not correct, more on that below.

Also, Porter claims to support AddPath, a mechanism that add multiple concurrent paths for a single route. There seems to be confusion in the use of this feature, started by a MetalLB post suggesting that this might be a mechanism to better balance how traffic is distributed to nodes that have multiple PODs. This is not the purpose of addpath, it is used to provide stability of routing tables where multiple paths would normally be summarized. If their are implementations that use addpath to add more ECMP next-hops to the same node, its likely that its a behavior specific to a particular vendor. Either way, most BGP implementations show addpath counts, we did not see porter-manager updating those counts on the upstream router to reflect the number of PODs on a node.

The BGP process runs as part of the porter-manager, therefore a failure of the porter manager results in the peer to the upstream router going down and the software router will withdraw all of the routes it learned from porter.

To address the problem caused by two routers on one node, porter connects to the upstream router with a different port number. It solves part of the problem, care must be taken to manually configure the router-id, not create a duplicate by using the host IP address.

## Traffic Policy

Changing externalTrafficPolicy:local in Porter does change its operation. Porter always sends traffic to a node or nodes with PODs. This is in violation of the behavior described by kubernetes. In BGP mode all services should be set to externalTafficPolicy:Local because this is the way Porter behaves irrespective of this configuration. This setting will result in kube-proxy being correctly configured, removing an unnecessary NAT step and preserving the source IP address. Porters Layer2 operation with multiple POD results in duplicate IP addresses, therefore further discussing how this setting would impact the existing incorrect network behavior is irrelevant.

## Configuration

Configuration is via custom resources, each EIP contains an address range and the associated protocol.

There is no further configuration required for L2, however there are two CR for BGP, BgpConf and BgpPeer. Further highlighting the limited implementation of BGP, BgpConf requires that the RouterID be defined. By convention this would be an address on host device. However this would be confusing as Kubernetes schedules this POD therefore another address should be used as static identifier. Unlike most implementations Porter does not have a mechanism to automatically select a valid address.

While discussing configuration, Porter has very limited documentation. It can be installed and configured from the documentation however if your evaluation requires that you figure out how it works, prepare to read golang code.

## IPv6

Porter has no support for IPv6. Scanning the code did not show that any IPv6 support is present.

## Conclusion.

Building these load-balancer orchestrators is not easy! K8s networking combined with host and routing protocols requires significant knowledge and skill. Therefore I commend the developers on their work. However, Porter suffers from feature and operational problems that would make it difficult to select as a solution unless you are willing to do a lot of additional development.