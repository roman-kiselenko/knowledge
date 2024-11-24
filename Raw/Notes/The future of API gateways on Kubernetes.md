---
title: The future of API gateways on Kubernetes
source: https://www.cncf.io/blog/2023/08/14/the-future-of-api-gateways-on-kubernetes/
clipped: 2023-09-03
published: 
category: clouds
tags:
  - k8s
  - network
  - gateway
read: true
---

By Pubudu Gunatilaka August 14, 2023

*Guest post originally published on [WS02’s blog](https://wso2.com/library/blogs/the-future-of-api-gateways-on-kubernetes/) by Pubudu Gunatilaka*

![[Raw/Media/Resources/cd60e53b96ed79ac501a05b24f8e661d_MD5.jpg]]

## Key Takeaways 

-   API gateways have evolved from basic converters to advanced cloud gateways, adapting to market demands and playing a crucial role in managing and securing API communications between clients and backend services.
-   Even as the open source Kubernetes Gateway API project positions API gateways to become an essential but commoditized part of infrastructure, full lifecycle API management will remain critical in driving cloud native applications and services.
-   To fully harness the potential of cloud native computing, traditional API management architectures must be redesigned to not only accommodate the requirements of Kubernetes and cloud native environments but also fully capitalize on the capabilities they enable.

## Introduction 

The exponential growth of the Internet and cloud computing has given rise to applications that are smaller, more distributed, and designed for highly dynamic environments capable of rapidly scaling up or down as needed. These applications have pushed the demand for modern API management product architectures that can leverage cloud native capabilities to achieve scalability, resilience, agility, and cost efficiency. At the same time, cloud native applications have helped push API gateways to evolve from simple converters to advanced intermediaries that are indispensable in driving digital initiatives. 

Meanwhile, [Kubernetes](https://kubernetes.io/) has emerged as the open source containerized orchestration system of choice for modern distributed applications, whether deployed on one or more public or private clouds. According to the 2022 annual Cloud Native Computing Foundation (CNCF) [survey](https://www.cncf.io/reports/cncf-annual-survey-2022/#findings), 89% of CNCF end-users are either using or piloting/exploring Kubernetes. The [Kubernetes Gateway API Project](https://gateway-api.sigs.k8s.io/) is a community effort to facilitate organizations’ implementations by defining a declarative API to configure gateways and ingress in Kubernetes clusters. This development is expected to make API gateways more accessible and commoditized even as full lifecycle API management remains critical in driving cloud native applications and services.

In this article, we will examine how API gateways have evolved, how traditional API gateways fall short in supporting Kubernetes cloud native environments, today’s API gateway requirements, why Envoy is the best standard on which to build next-generation API gateways, and the future of API management on Kubernetes.

## The Evolution of Gateways

![[Raw/Media/Resources/1bf2f2d7b2f4b89a9c44dae2eff0cf1f_MD5.png]]

*Figure 1: The evolution of gateways*

Before examining how API gateways have evolved, let’s quickly define what we mean by API management versus an API gateway. API management encompasses the end-to-end process of strategically planning, designing, implementing, and monitoring APIs at every stage of their lifecycle. An API gateway is a dedicated component within the API management infrastructure that serves as the primary entry point for API requests, facilitating communication between clients and backend services. 

The evolution of API gateways can be categorized into nine distinct stages, each representing advancements and changes in their capabilities. These stages reflect the continuous development and adaptation of API gateways to meet the evolving needs of modern application architectures and API management.

-   ***Early gateways*** functioned as converters, enabling communications between different network architectures and facilitating connectivity among diverse systems.
-   ***Proxy servers,*** including Apache HTTP Server and Squid, gained popularity as gateways, managing traffic across multiple endpoints and providing features, such as caching, security, and access control.
-   ***Hardware load balancers,*** such as F5 Networks Big-IP emerged to optimize performance and handle increasing user demands as applications became more complex.
-   ***Software load balancers*** included HAProxy, which emerged as a popular choice due to its robust features, performance, and reliability. It provided advanced load balancing algorithms, session persistence, health checks, and scalability options, making it well-suited for handling high volumes of API traffic.
-   ***Application delivery controllers*** (ADCs) like NGINX and Citrix ADC emerged as advanced gateways for optimizing application performance and ensuring secure communications between clients and servers. They offer load balancing, Secure Sockets Layer (SSL) and Transport Layer Security (TLS) termination, security, caching, and traffic management capabilities
-   ***SOAP gateways*** naturally evolved alongside Simple Object Access Protocol (SOAP) and web services APIs, enabling integration, protocol conversion, and security enforcement for SOAP-based services.
-   ***API gateways*** evolved with the rise of web APIs and service-oriented architectures to serve as intermediaries for accessing APIs and handling tasks, such as routing, authentication, rate limiting, and transformation. They play a vital role in securing, monitoring, and managing API traffic, with some leveraging the Netty networking framework. API gateways have further evolved to handle the complexities of communication and routing in microservices architectures, integrating with service discovery mechanisms for dynamic routing, load balancing, and protocol translation.
-   ***Microgateways*** are increasingly used at the network edge to support the rise of microservices, edge computing and the Internet of Things (IoT). They bring processing power and connectivity closer to data sources, reducing latency and congestion while enabling data aggregation, local processing, and security enforcement. They also facilitate communication between IoT devices and backend systems.
-   ***Cloud native API gateways*** have emerged to manage traffic in cloud native environments, utilizing container orchestration platforms like Kubernetes and embracing cloud native principles for scalability and resilience. With the focus on automation and enhanced intelligence in cloud native environments, gateways increasingly incorporate artificial intelligence (AI) and machine learning (ML) techniques.

## The Role of an API Gateway 

![[Raw/Media/Resources/17d07d6196dff6796c42170329d4fce8_MD5.png]]

Figure 2: Overview of an API gateway

The role of the API gateway in today’s technology landscape has been significantly shaped by its evolution. Presently, the API gateway, which is at the heart of API management, acts as the entry point for API requests, abstracting quality of service (QoS) concerns from individual services. It consolidates various QoS features, including security, rate limiting, message transformation, request routing, caching, and API insights to ensure the comprehensive management of API requests. 

***API security*** is crucial, with authentication and authorization playing key roles. Authentication methods, such as mutual TLS (mTLS), Open Authorization (OAuth) 2.0 tokens, and API keys, along with fine-grained access control through authorization scopes, enhance security. 

***API rate limiting*** ensures fair usage among users by adhering to defined policies. It prevents abuse, maintains a consistent quality of service, and promotes transparency and accountability in API management.

***Caching*** offers significant benefits for API-driven applications. It enhances performance, scalability, and resilience by reducing the backend load, minimizing latency, optimizing bandwidth usage, and providing resilience to backend failures.

***Message transformation*** capabilities allow users to modify API requests, but implementing extensive transformations directly at the API gateway can impact performance. The best practice is to utilize an integration layer designed for intricate message transformations. 

***Request routing*** involves directing incoming API requests to the appropriate backend services based on defined rules and configurations, while also incorporating service load balancing, failover mechanisms, and A/B testing.

***API insights*** provide business intelligence by collecting and publishing data for analysis and visualization. 

***Monitoring and observability*** involve data collection and analysis to gain insights into API performance, availability, and usage. These functions allow organizations to track metrics, detect anomalies, and troubleshoot issues, ensuring the efficient operation and continuous improvement of the API gateway and the underlying API infrastructure.

Modern API gateways go beyond support for Representational State Transfer (REST) services to also include support for GraphQL services, gRPC services, and various streaming services. Each type of API receives different QoS treatments from the API gateway. Deployment patterns include the centralized pattern, where all APIs are scaled together, and the decentralized pattern, where APIs are distributed across multiple gateways based on factors like criticality and operational needs. The API manager control plane manages gateways deployed in different environments, providing centralized control and configuration.

## API Management in a Cloud Native Landscape

The adoption of Kubernetes has driven the increased use of API gateways as organizations leverage the benefits of containerization and microservices architectures. API gateways enable efficient API management, scalability, and security in Kubernetes-based environments, making them a vital component in modern application development and deployment.

### Gaps in Using Traditional API Management for Kubernetes

Despite their wide use, traditional API management architectures are not well-suited for Kubernetes and other cloud native environments. They have a monolithic design that bundles multiple functions together in one application. This makes it hard to scale individual components independently. Instead, the whole application has to be scaled together, which wastes resources and leads to performance issues when handling high traffic or increased workloads. 

Traditional API management architectures also depend on specific infrastructure components and technologies that are incompatible with cloud native environments. Not only do they have larger memory footprints and longer server starting times; they also may need dedicated hardware, specific software configurations, or proprietary solutions. This limits their ability to use the benefits of cloud platforms, including auto-scaling, infrastructure as code practices, and multi-cloud deployment.

API gateways face similar challenges in supporting cloud native applications due to some external dependencies in API management architectures. To overcome these challenges and meet the needs of both current and future cloud native environments, the API gateway needs to be re-architected.

### Envoy Proxy: The Key to Re-architecting API Gateways

[Envoy](https://www.envoyproxy.io/) has emerged as the preferred solution for re-architecting API gateways, since the open source edge and service proxy has been designed specifically for cloud native applications. Additionally, its adaptability, scalability, and strong security features make it an excellent choice for managing API traffic in diverse cloud environments. Furthermore, as an open source solution, Envoy has received continuous improvements from developers worldwide, who have contributed new features, enhancements, and bug fixes, further enhancing the proxy’s robustness and reliability. 

***From an API management perspective***, Envoy offers essential features, such as rate limiting, response caching, cross-origin resource sharing (CORS) handling, request authentication, and authorization. It supports the ability to expose both REST and gRPC services, and it has direct integrations with Amazon Web Services (AWS) for exposing AWS Lambda functions. The proxy also provides built-in observability features that seamlessly integrate with open telemetry, offering comprehensive metrics and trace data for monitoring and debugging. The Envoy proxy is configured using xDS APIs, which enable dynamic configuration, service discovery, traffic routing, and runtime control of the proxy. By leveraging the xDS APIs, Envoy can be easily set up and modified to meet evolving requirements. 

***An Envoy proxy is deployed in two main patterns:*** the sidecar proxy pattern and the front proxy pattern. In the sidecar proxy pattern, Envoy serves as a communication bus for internal traffic in service-oriented architectures (SOA). It is commonly used as a sidecar in service meshes, managing network traffic alongside the services it proxies. Envoy’s lightweight core and robust features, such as load balancing and service discovery, make it a popular choice for service mesh implementations, such as Istio.

In the front proxy pattern, Envoy is utilized in Kubernetes Ingress controllers to expose services to external consumers. Some Ingress controllers are built on top of Envoy, leveraging its essential capabilities. Envoy’s flexible configuration model and feature set also make it an ideal fit for API gateways. The majority of modern API gateways are built on top of the Envoy proxy, which serves as the fundamental framework.

### Cloud Native API Management Architecture

API gateways based on the Envoy proxy have a crucial role in enhancing the overall API management architecture since they leverage cloud native technologies and practices to effectively manage APIs while ensuring scalability and resilience. Here, the API management architecture is built on microservices principles, containerization, and container orchestration using Kubernetes, providing the essential infrastructure for efficient API management in a cloud native environment. 

A cloud native API management architecture comprises two distinct planes: the control plane and the data plane. The control plane is the command center where API management tasks and API key generation are performed. The data plane serves as the gateway for API calls, exposing the created APIs to public or private consumers. The architecture is designed to be highly scalable, allowing for a single control plane to govern multiple data planes. This enables seamless API gateway federation and facilitates API management across various cloud providers or on-premises data centers. To enhance scalability and resilience, the architecture leverages fundamental features of Kubernetes, such as auto scaling and auto healing, to ensure optimal performance and reliability.

![[Raw/Media/Resources/5bb509753a86ad62d3dab6b9e75482b1_MD5.png]]

Figure 3: The control plane and data plane for API management

## The Envoy Gateway Project and API Standardization

In 2022, Matt Klein, the creator of Envoy, introduced a new project called [Envoy Gateway](https://gateway.envoyproxy.io/), specifically targeting API gateways. Envoy already had the necessary components for building an API gateway, including a proxy layer; a configurable filter architecture for network traffic filtering, routing, and processing; and xDS APIs for transmitting data to Envoy proxies. The open source Envoy Gateway project adds a management layer to handle an Envoy proxy as a standalone gateway or as a Kubernetes-managed API gateway. 

The project’s goal is to offer a streamlined API layer and deployment model, focusing on simpler use cases. It also aims to consolidate various CNCF API gateway projects into a common core, enabling vendors to build value-added solutions on top of Envoy and Envoy Gateway. By fostering community collaboration, the project also aims to eliminate duplicative efforts in developing fundamental API gateway features, allowing vendors to concentrate more on API management capabilities. This unification effort holds the potential to enhance efficiency and encourage innovation within the API gateway space.

The Envoy Gateway project is primarily built on the [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/), which defines how Kubernetes services can be exposed to the external world. It leverages Kubernetes Custom Resource Definitions (CRDs), such as GatewayClass, Gateway, HTTPRoute, TCPRoute, and others to configure an API gateway. The CRDs outline important features like traffic splitting, request routing, redirects, rewrites, and TLS handling. Moreover, various policies can be associated with the API gateway or specific API paths. 

By adopting the Kubernetes Gateway API as the foundation, vendors have the flexibility to develop their own implementations on their preferred technology stack, and they can utilize extension points provided by the API specification to incorporate vendor-specific functionality. This approach fosters the adoption of a shared, standardized API specification that facilitates interoperability, promotes consistency across API gateways, and opens up new possibilities for future innovations. 

## The Future of API Management on Kubernetes

API management plays a vital role in the dynamic API landscape, and as API standardization advances, it’s inevitable that the API gateway will become a fundamental element of the infrastructure. This move will free developers from the burden of directly managing the gateway and its associated complexities, allowing them to instead devote their attention to constructing and maintaining APIs and other core development tasks. Consequently, the focus shifts towards API management, which encompasses a range of essential features. 

-   ***API lifecycle management*** enables the provisioning of lifecycles for APIs, allowing for custom events and stakeholder authorization of lifecycle changes.
-   ***API governance*** establishes processes and policies to ensure effective API development, management, and compliance with organizational goals and standards.
-   ***An API marketplace*** serves as an online platform for developers to discover, access, and manage APIs. It offers documentation, catalogs, software development kits (SDKs), testing tools, community support, and monetization options.
-   ***API version management*** involves managing different API versions to accommodate changes while maintaining backwards compatibility.
-   ***API productization*** treats APIs as marketable products, applying product management principles to meet developer and business needs. This includes creating a user-friendly experience and implementing monetization strategies.
-   ***API insights*** provide valuable information by analyzing usage, performance, and behavior, enabling data-driven decisions for enhancing the API experience. 

To fully leverage the benefits of the cloud native ecosystem, many of these features will need to be redesigned. This will involve seamless integration with various third-party services available on Kubernetes to enable a more comprehensive and robust API management solution.

Furthermore, the adoption of an open source distribution model will be a crucial factor for enterprises. Open source products encourage community participation and adaptation, fostering collaboration and innovation within the ecosystem. Embracing open source also ensures that an API management solution can benefit from the collective knowledge and contributions of a diverse community, leading to continuous improvement and evolution.

## A Single Control Plane for API Gateways

![[Raw/Media/Resources/c5171d202e9d2e8324ef3b5c4b8266f4_MD5.png]]

Figure 4: A single control plane for different API gateway implementations

API standardization transforms the API gateway into a commodity, paving the way for a unified control plane that encompasses API management and discovery. Simultaneously, the data plane can consist of one or more API gateway implementations from various vendors. This paradigm shift presents numerous implications and opportunities.

-   ***Vendor neutrality and flexibility:*** Organizations attain vendor neutrality and enhanced flexibility by utilizing a single control plane capable of managing multiple gateways from diverse vendors. This empowers them to select the most appropriate gateway solution for each unique use case or environment. By embracing multiple vendors, organizations can harness the specific strengths and advantages offered by different providers, avoiding vendor lock-in and fostering a more adaptable ecosystem.
-   ***Interoperability and standardization:*** The adoption of a single control plane fosters interoperability and standardization among diverse gateways. By adhering to shared APIs, protocols, and data formats, the control plane enables seamless communication and management of gateways from various vendors. This promotes consistency in configurations, policies, and security measures, leading to improved stability and reliability across the API infrastructure.
-   ***Ecosystem expansion and innovation:*** The ability of a single control plane to manage gateways from multiple vendors fosters a diverse ecosystem and stimulates innovation. It enables organizations to embrace and integrate new gateway solutions from emerging vendors without causing disruptions to their existing infrastructure. This healthy competition among vendors drives innovation, expands the range of available options, and empowers organizations to meet their API management needs with a broader selection of cutting-edge solutions.
-   ***Cost optimization:*** The availability of a single control plane to manage gateways from various vendors grants organizations the flexibility to select cost-effective gateway solutions that align with their specific requirements and budgetary considerations. This freedom enables enterprises to optimize their spending, extract maximum value from their API management infrastructure, and make well-informed decisions that align with their specific needs.
-   ***Simplified management and operations:*** Organizations can centrally configure, monitor, and control their API gateways, eliminating the complexities associated with managing disparate gateways individually. This centralized approach enhances administrative efficiency, streamlines operations, and minimizes potential inconsistencies. As the gateways follow the same API standardization, migration between API gateway solutions becomes easier. This simplifies the process of transitioning from one gateway solution to another, ensuring smooth integration and reducing the potential disruption to existing infrastructure.

In summary, when the API gateway becomes a commodity and a single control plane manages multiple gateways from different vendors, organizations gain the benefits of vendor neutrality, simplified management, greater interoperability, cost optimization, enhanced security, and easier ecosystem expansion. The convergence will also empower enterprises to leverage the strengths of diverse gateway solutions, streamline operations, drive innovation, and optimize the practices around their API management.