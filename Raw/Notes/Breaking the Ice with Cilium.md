---
title: Breaking the Ice with Cilium
source: https://itnext.io/cilium-cni-a-comprehensive-deep-dive-guide-for-networking-and-security-enthusiasts-588afbf72d5c
clipped: 2023-09-03
published: 
category: clouds
tags:
  - k8s
  - service-mesh
  - network
read:
---

> *Breaking the Ice with Cilium*

Welcome to our comprehensive deep-dive guide for networking and security enthusiasts!

In this blog post, we will break down the concepts and intricacies of `Cilium` in a way that even beginners can understand. So, if you’ve been curious about how Cilium can enhance your network’s performance and security, you’ve come to the right place!

Cilium, a powerful Container Network Interface (CNI) plugin, has emerged as a leading solution, revolutionizing networking and security in the cloud-native landscape.

But what exactly is Cilium, and how does it work? Fear not, as we’ll guide you through this journey step by step, using plain and easy-to-understand language. Whether you’re a developer, a system administrator, or simply someone fascinated by cutting-edge networking and security technologies, this guide will provide you with a solid foundation to start exploring Cilium’s capabilities.

Throughout this blog post, we’ll cover the basics of Cilium, including its core features, how it integrates with container orchestration systems like Kubernetes, and how it leverages eBPF (extended Berkeley Packet Filter) technology for enhanced network visibility and security. By the end, you’ll have a clear understanding of how Cilium can help you build robust, scalable, and secure networking solutions for your applications.

So, if you’re ready to dive into the world of Cilium, let’s break the ice and embark on this exciting journey together!

Cilium addresses various use cases related to what is known as Service Mesh. In this section, we will explore what it entails, how to configure it in Cilium, and the benefits of managing these elements through Cilium’s eBPF.

[**Ingress:**](https://docs.cilium.io/en/stable/network/servicemesh/http/)

Cilium can serve as an `Ingress Controller`, managing Ingress if configured accordingly (the documentation provides details on the different parameters to enable based on your requirements).

Deploying Cilium’s capabilities as an Ingress Controller allows you to define an Ingress, just like any other Ingress Controller. Simply specify ingressClassName: cilium in the specification to rely on Cilium.

Going beyond simple Ingress, Cilium also supports the relatively new `Gateway API`. 🎉 \[ my best feature 😃\]

[**Gateway API:**](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/) **🚪**

![[Raw/Media/Resources/b52e88df03429d62e8b5b490828dba80_MD5.png]]

[The Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) is a specification for different Kubernetes resources that extend the capabilities of a cluster by offering an interface that can be implemented differently by various K8s service providers, such as Cilium.

Therefore, it is a standard to follow to enable easier interoperability of external services with a Kubernetes cluster.

Once the service is enabled in the cluster and in Cilium’s Helm Chart ( **— set gatewayAPI.enabled=true**), it becomes possible to create resources that comply with the Gateway API specification (**Gateway, HTTPRoute, etc.)**:

![[Raw/Media/Resources/8086cfdda3e94b256efac280b0408df6_MD5.png]]

Cilium now offers a fully conformant implementation of the Gateway API, which is the new standard for `North-South Load-Balancing` and Traffic Routing in Kubernetes clusters. While maintaining and expanding Ingress support, Gateway API represents the future of traffic management.

The development of Gateway API was driven by the `limitations of the Ingress API`. It lacked advanced load-balancing capabilities and only supported basic content-based request routing for HTTP traffic. Additionally, managing Ingress became challenging as vendors relied on annotations to address its functionality gaps, leading to inconsistencies.

To overcome these limitations, the Gateway API was designed from scratch. It addresses core routing requirements and incorporates operational lessons learned from using Ingress resources for several years. The API follows a role-oriented approach, allowing network architects to build shared networking infrastructure for multiple teams without coordination issues. 🕸

✅ In Cilium 1.13, the Gateway API passes all conformance tests and supports various use cases such as:

-   *HTTP Routing*
-   *TLS Termination*
-   *HTTP Traffic Splitting / Weighting*
-   *HTTP Header Modification*

Cilium, powered by [Envoy](https://www.envoyproxy.io/) technology, offers advanced `[L7 traffic management](https://docs.cilium.io/en/stable/network/servicemesh/l7-traffic-management/)` capabilities within a service mesh architecture.

It allows developers and operators to define Envoy configurations at the namespace or global level, enabling features such as load balancing, traffic splitting, mirroring, manipulation, and encryption. With the right configurations enabled, Cilium empowers users to optimize performance, enhance security, and gain valuable insights into their L7 traffic.

![[Raw/Media/Resources/66dc19e77a6f435b71d7ec729ee9841a_MD5.png]]

In Cilium 1.13, a new feature was introduced to achieve L7 load balancing for Kubernetes services using Cilium’s embedded Envoy proxy. By applying a simple annotation `service.cilium.io/lb-l7: enabled` to a Kubernetes service, L7 load balancing can be enabled seamlessly within the cluster, including multi-cluster scenarios with Cilium ClusterMesh.

This feature addresses the operational complexity of `[gRPC load balancing](https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/)` and eliminates the need for additional tools or service meshes.

Additionally, Cilium provides improved functionality for Ingress API Resources. In version 1.13, Cilium Ingress supports Shared LoadBalancer mode, where multiple Ingress resources share the same LoadBalancer resource and IP address. This significantly reduces costs associated with cloud load balancers and public IPs, providing evident cost benefits for cloud engineers.

These features highlight the powerful capabilities of Cilium in managing L7 traffic, integrating with Envoy, and providing efficient load-balancing solutions for Kubernetes services.

Ensuring the security of traffic is a fundamental aspect of any software architecture. Cilium, as a `Container Network Interface (CNI)`, recognizes the significance of security while maintaining a simple and expressive configuration experience.

To enforce security, Cilium utilizes network policies, such as `**CiliumNetworkPolicy**`, which allows for the restriction of incoming traffic exclusively from pre-approved sources. By applying these policies, outgoing traffic from endpoints labeled with `org=myorg` can be confined to a specific `Fully Qualified Domain Name (FQDN)` like `secure.my-corp.com.` This ensures that no other traffic is allowed to leave these endpoints and communicate with external FQDNs.

It’s important to note that applying a network traffic policy to egress/exit transitions the endpoint from `Default Allow` to `Default Deny` mode, offering a controlled environment.

In the context of network policies, considerations are made for the `kube-system/kubedns` and ***port 53***. These considerations are necessary to ensure that pods within the endpoint effectively utilize DNS resolution. Since applying the mentioned policy transitions the endpoint to `Default Deny` mode, it becomes imperative to explicitly allow the endpoint to send DNS queries.

Additionally, Cilium 1.13 introduces support for `Server Name Indication (SNI)` in Network Policies, further enhancing network security. SNI is an extension of the `Transport Layer Security (TLS)` protocol that enables multiple domain names to be served by a single IP address.

With this new feature, operators can now restrict the allowed TLS SNI values within their network, fostering a more secure environment. Implementation-wise, SNI support is achieved through Envoy redirect, redirecting connections to different endpoints based on the SNI value present in the client’s message.

To leverage the SNI support in Cilium, a new field called `***ServerNames***` has been added to the Cilium Network Policy. This field allows operators to specify a list of permitted TLS SNI values. If the `***ServerNames***` field is not empty, TLS must be present, and one of the provided SNIs must be indicated during the TLS handshake.

With the following policy, you can allow traffic to the `amit.cilium.rocks` SNI:

![[Raw/Media/Resources/a1447f25ea9087ace51882dc16f9e57f_MD5.png]]

The widespread use of TLS encryption, especially through protocols like HTTPS, has become a cornerstone of data security in our daily interactions. The security guarantees provided by such protocols ensure the protection of valuable data entrusted to these connections.

One might think that encryption poses challenges to the management and observability of network traffic, as it renders connections opaque. However, we will explore here that this is not the case, thanks to the mechanism of `[TLS inspection](https://docs.cilium.io/en/stable/security/tls-visibility/)`, which closely resembles a “Man-in-the-Middle” attack orchestrated by Cilium.

![[Raw/Media/Resources/2836e684e33a9db17e4f80cb1d481562_MD5.png]]

Instead of having a typical egress flow, Cilium is capable of performing the following steps:

1.  Intercepting a newly created request from a Cilium endpoint. This request is generated using an internal certificate authority recognized by Cilium.
2.  Performing TLS termination, thereby gaining access to the content of the request.
3.  Executing network policies and other relevant actions.
4.  Recreating a new TLS request using public certificate authorities (or at least those approved for outbound traffic within your organization’s cluster).
5.  Issuing the recreated HTTPS request to the original target, which then performs its standard termination process.

Through this process, Cilium enables the `**inspection of TLS connections**` without compromising security. It leverages the ability to intercept and recreate requests, allowing for effective network policy enforcement and observability within the encrypted traffic.

By employing TLS connection inspection, Cilium strikes a balance between secure encryption and the necessary management and monitoring of network traffic, ensuring both data protection and operational visibility.

Mutual authentication is highly valued by users of service meshes, who see it as a crucial feature. They have been eagerly anticipating the implementation of this feature in Cilium Service Mesh. In Cilium 1.13, we are introducing support for mTLS (mutual Transport Layer Security) at the datapath level. This enables Cilium to authenticate endpoints on peer nodes within the cluster and control data plane connectivity based on successful mutual authentication.

While this implementation may not have immediate user-facing changes, it lays the groundwork for future developments to make mTLS ready for users. We recently published a detailed post outlining our vision for mutual authentication with Cilium Service Mesh, and we are excited to share our progress with the release of Cilium 1.13.

Existing mutual authentication implementations often impose various restrictions on users to achieve enhanced security. These restrictions can include using TCP+TLS for authentication and encryption or requiring the use of (sidecar) proxies in the data plane. However, such restrictions increase complexity and processing costs. We believe that by combining the strengths of session-based authentication and network-based authentication, we can offer a secure implementation that is faster and more flexible than previous approaches.

Our long-term goals include providing a mechanism for pods to opt into peer authentication, supporting pluggable certificate management systems (such as SPIFFE, Vault, SMI, Istio, cert-manager, etc.), offering configurable certificate granularity, and leveraging existing encryption protocol support for the data plane.

![[Raw/Media/Resources/a1899905f49ec1c9a4e1d18c2a89b5d4_MD5.png]]

In a [collaboration](https://grafana.com/about/press/2022/10/24/grafana-labs-partners-with-isovalent-to-bring-best-in-class-grafana-observability-to-ciliums-service-connectivity-on-kubernetes/?pg=blog&plcmt=body-txt) between `Grafana and Isovalent`, Cilium integration with Grafana was announced, enabling comprehensive monitoring of network connectivity between cloud-native applications in Kubernetes. This integration utilizes `Prometheus metrics,` `Grafana dashboards` and alerts, and Grafana Tempo traces to observe the health and performance of the network.

With the release of Cilium 1.13, the partnership brings significant advancements. One notable improvement is the ability to troubleshoot distributed systems by incorporating tracing. Previously, application developers could send traces to their preferred tracing backend, such as `[Grafana Tempo](https://grafana.com/oss/tempo/)`. However, network metrics were not included. Now, with Cilium 1.13, this gap is bridged.

`**Hubble’s Layer 7 HTTP visibility feature**` in Cilium 1.13 extracts OpenTelemetry traceParent headers from Layer 7 HTTP requests and links them to Hubble flows. This integration allows users to see their application’s traceIDs in Hubble’s L7 HTTP metrics, providing a seamless transition from metrics to traces.

Additionally, Cilium 1.13 introduces the Hubble Datasource Plugin, expanding Grafana’s ecosystem. This plugin enables detailed insights into network traffic, allowing correlation with application-level telemetry. It integrates with Prometheus for storing Hubble metrics, Tempo for storing distributed traces, and Hubble Timescape, the Isovalent Cilium Enterprise observability platform.

With these advancements, users can leverage Grafana’s capabilities to gain a comprehensive view of their data sources, achieving enhanced network observability and correlation between network traffic and application telemetry.

![[Raw/Media/Resources/2683fb168a06fd934fd72320871f65d0_MD5.png]]

Cilium, a `favored choice among cloud providers, financial institutions, and telecommunications providers`, is gaining widespread adoption due to its ability to maximize network performance. These organizations share a common goal of seeking incremental gains in network performance, especially as they build networks capable of handling 100Gbps and beyond.

However, the adoption of high-speed **100Gbps** network adapters brings forth a challenge: How can a CPU effectively handle the immense volume of packets, which can reach up to eight million packets per second with an assumed MTU of 1,538 bytes? With a mere 120 nanoseconds available for the system to process each packet, it becomes clear that the existing approach is unrealistic. Additionally, a substantial overhead is involved in managing the interface-to-upper protocol layer for handling such an enormous packet load.

Enter `**BIG TCP**`, a groundbreaking solution. What if packet payloads could be grouped together into super-size packets? By increasing the packet size, the overhead can be reduced, leading to potential improvements in throughput and latency reduction.

With the latest release of Cilium, version 1.13, the exciting feature of BIG TCP is now available for utilization in your cluster, offering enhanced performance benefits.

Cilium CNI is a powerful networking and security solution for Kubernetes. Its eBPF-based data plane offers high performance and flexibility. With features like encryption, load balancing, and distributed firewalling, it enables secure microservices architectures.

Cilium’s observability and security capabilities provide deep insights into network traffic and protection against attacks. Its integration with Kubernetes RBAC allows granular security policies.

For networking and security enthusiasts, exploring Cilium opens up exciting possibilities in Kubernetes deployments. Dive in, break the ice, and elevate your networking and security game with Cilium CNI!

If you found this article helpful, please **don’t forget** to hit the **Follow** 👉 and **Clap** 👏 buttons to help me write more articles like this.  
Thank You 🖤

Thank you for Reading !! 🙌🏻😁📃, see you in the next blog.🤘

🚀 Feel free to connect with me :

**LinkedIn**: [https://www.linkedin.com/in/rajhi-saif/](https://www.linkedin.com/in/rajhi-saif/)

**Twitter** : [https://twitter.com/rajhisaifeddine](https://twitter.com/rajhisaifeddine)