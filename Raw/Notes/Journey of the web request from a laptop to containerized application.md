---
title: Journey of the web request from a laptop to containerized application
source: https://medium.com/@olexandr.pochapskiy/journey-of-the-web-request-from-a-laptop-to-containerized-application-9f6ea4211bb9
clipped: 2023-09-04
published: 
category: development
tags:
  - network
  - low-level
read: false
---

Photo by [Jordan Harrison](https://unsplash.com/@jordanharrison?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral)

I’m currently heavily invested in learning internals of the cloud computing and one of the interesting topics for me was how the whole AWS network stack is set up so that we can expose our applications to the internet. And so I decided to research this area deeper and complement it with networking related to actually getting to the AWS from my laptop

The topic of this post is centered around the journey of a network packet as it travels from a laptop toward an Nginx application that is hosted within AWS EKS with the VPC CNI plugin included, and is load balanced by the infrastructure set up by AWS ALB Controller.

![[Raw/Media/Resources/515994c6fa8b45c4fe8b2125af49ff2e_MD5.png]]

Routing from laptop to AWS Autonomous system

To begin, when our laptop connects to the local network, the DHCP client daemon initiates a broadcast DHCP request to acquire information regarding the IP address, default gateway, DNS server, and other network settings. Local routing tables get updated with the default gateway as a result of this operation. The default gateway is the address to send any non-local request and the DNS server refers to the server’s address responsible for converting domain names into corresponding IP addresses.

After configuring all network settings, we enter our example domain *packet-journey.com* into the address bar of our web browser. If we are accessing the domain for the first time we need to go through the DNS Lookup procedure to identify the IP where to send the packet. Our domain *packet-journey.com* is set up via CNAME DNS record means it is an alias for an AWS-generated ALB domain. That is why during the IP resolution procedure DNS resolver might send two requests to the recursive DNS server — one for the CNAME record (will resolve *packet-journey.com* to ALB domain) and another one for A record (will resolve ALB domain to IP). The primary function of a recursive name server is to locate the authoritative name server that contains a domain record and retrieve the associated IP address. For further requests to the *packet-journey.com* to avoid full network roundtrip to the name server lookup result is cached in the browser itself, in OS, or/and on the router, we use to access the internet

Once the browser receives the target IP address, it initiates the process of establishing a secure TCP connection with the server. It starts with a 3-way handshake process followed by application data transmission. Routing is the same for handshake and data transfer, the only difference is that the 3-way handshake process ends at the ALB level and data transfer goes all up to nginx. To put it simply browser creates a socket, which serves as a communication endpoint, and proceeds to send encrypted data over this connection. The local routing table is then used to identify the next hop for the packet destined outside of the local network and as was mentioned previously default gateway address is used for this. The default gateway address directs network traffic to the local router, which maintains its own routing table. Within this routing table, another default gateway is set to route traffic to the ISP's Autonomous System. An Autonomous System is a network utilizing part of the global IP prefix range. The Internet is just a set of such Autonomous systems that exchange information about each other IP prefix range with BGP protocol. Once the packet enters the ISP AS and it lacks knowledge of the AS responsible for the prefix range of the target IP, it will be passed along to various ASes in an attempt to locate the one that has the appropriate route to the AWS AS.

This route will then be used to get into AWS Autonomous System…

![[Raw/Media/Resources/2d8a24299d044efd81b29e75858d6951_MD5.png]]

ALB routing to pods

To understand how request goes through ALB components it is worth explaining how ALB was set up. If we want to expose some Kubernetes services to the external world, we use ALB Controller with Ingress resource. ALB controller ensures that ALB components are set up and up to date with the Ingress resource specification. Ingress resource is a Kubernetes object that defines how to route traffic from ALB to backend Kubernetes services. In practice, ALB Controller listens to the changes of Ingress resources via the Kubernetes API Server and automatically creates/updates ALB components like listeners, rules, and target groups. Let's briefly describe those ALB components. Listeners are about ports and protocols we expect requests on. Rules are routers to different target groups based on path, headers, source IP, etc. And target groups are just set of instance IDs or IPs that will handle the request. Listeners and rules configuration are more static and taken from the Ingress resource itself but IPs in the Target group are changed with every pod schedule/remove if we are talking about the target group setup in IP mode. ALB Controller listens to pod creation events via Kubernetes API Server to extract the IP of the new pod and put it into the target group for load balancing. When the pod is removed from the instance its IP is also removed from the target group.

Alright, now that we have discussed all the components that play a role in the ALB setup, let’s see what the actual flow of traffic looks like.

Our ALB which ALB Controller created has a configured listener on port 443 and can receive our HTTPS traffic. The listener performs SSL/TLS termination, which decrypts traffic for the downstream components. That allows unloading this compute-heavy operation from backend servers. The listener then routes the traffic via the only rule we have “ /\*” that handles all requests toward the target group. The Target group is where the magic of load balancing happens. Our target group configuration defined with IP mode means we are sending requests not to the instances but to pod IPs directly. The default load-balancing strategy here is “least outstanding requests” so a request is sent to the IP of the pod with the lowest number of requests it is currently processing.

The question is — how can we directly send traffic to the pod IP and not to the instance IP? Enter container networking with the CNI!

![[Raw/Media/Resources/093f78f390bef935d43726d1f851978c_MD5.png]]

Routing with VPC CNI plugin

The CNI specification provides a standard interface for configuring container networking. In other words, it describes the protocol, format, and procedures containers communicate with. That allows platforms like Kubernetes to plugin CNI implementation suitable for specific environments. CNI plugins vary in performance characteristics, security, network policies, etc. Also, they can natively support the environment they are running in. Like AWS in our case. AWS VPC CNI Plugin is an implementation of CNI for Kubernetes. The plugin integrates with AWS infrastructure in several ways, including the allocation of IPs to pods from the VPC subnet IP address range, attachment of ENI to instances when pods require additional IPs and native support for security groups.

Isolation of the pod network in Kubernetes is done by creating a separate network namespace. But the pod-specific network namespace needs to link to the host namespace to receive any external requests. The plugin is running as a Daemonset on every node and when the new pod is scheduled on the node, the plugin links the host and pod namespaces for external traffic to get to the pod and vice versa. To achieve that plugin creates virtual interface pair with one end in the host namespace and another one in the new pod namespace. It also gets new IP from the pool it manages locally and assigns it to the virtual interface created in the pod network namespace. When all IPs of the instance are occupied plugin can allocate a new Elastic Network Interface, attach it to the instance, and have more IPs to be attached to the instance pods. Finally, the setup is completed by modifying the routing tables. This involves adding an entry that directs all traffic destined for the new IP address to the virtual interface pair, and consequently, to the pod.

So, as described above, when we enter the node we are looking for the entries in the routing table for our IP, we then traverse through the virtual network pair and enter the pod environment.

Our nginx container is part of the pod network namespace. That means sockets opened by nginx within the container are visible to the pod environment. So when our nginx container opens the socket on port 80 and the pod network interface receives incoming packets network stack queues them for the nginx to process. And that's it, the packet is delivered!

References: