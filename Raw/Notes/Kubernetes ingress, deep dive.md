---
title: Kubernetes ingress, deep dive
source: https://banzaicloud.com/blog/k8s-ingress/
clipped: 2023-09-04
published: 
category: network
tags:
  - k8s
  - network
  - ingress
read: false
---

We often find ourselved required to route traffic from external sources towards internal services deployed to a Kubernetes cluster. There are several ways of doing this, but the most common is to use the `Service` resource, or, for HTTP(S) workloads, the `Kubernetes Ingress API`. The latter is finally going to be marked GA in K8s 1.19, so let’s take this opportunity to review what it can offer us, what alternatives there are, and what the future of ingress in general could be in upcoming Kubernetes versions.

## How to expose applications in Kubernetes [🔗︎](#how-to-expose-applications-in-kubernetes)

Usually, we use the Service resource to expose an application internally or externally: define an entry point for the application which automatically routes distributed traffic to available pods. Since pods tend to come and go – the set of pods running in one moment in time might be different from the set of pods running that application at some later point – the Service resource groups them together with a label selector.

Service resources are broken down by type for more versatile usage. The three most commonly used types are `ClusterIP`, `NodePort` and `LoadBalancer`. Each provides a different way of exposing the service and is useful in different situations.

-   ClusterIP: exposes the Service on a cluster-internal IP. Choosing this value makes the Service only reachable within the cluster. This is the default ServiceType.
-   [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport): exposes the Service on each Node’s IP at a static port. A ClusterIP Service towards the NodePort Service route is created automatically. We’ll be able to access the NodePort Service from outside the cluster by requesting `<NodeIP>:<NodePort>`.
-   [LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer): exposes the Service externally using a cloud provider’s load balancer. NodePort and ClusterIP Services, towards the external load balancer routes are automatically created.

ClusterIP is typically used to expose services internally, NodePort and LoadBalancer to expose them externally. The `Service` resource essentially provides `Layer-4` TCP/UDP load balancing, simply exposing the application ports as they are.

> A `LoadBalancer`\-type service could also be used for `Layer-7` load balancing if the provider supports it through custom annotations.

The `Ingress` resource is the most commonly used way of getting more fine-grained `Layer-7` load balancing for HTTP protocol-based applications.

## What is Ingress? [🔗︎](#what-is-ingress)

### Ingress API [🔗︎](#ingress-api)

[Ingress](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#ingress-v1beta1-networking-k8s-io) is an API resource that provides a simple way of describing HTTP and HTTPS routes from outside the cluster to [services](https://kubernetes.io/docs/concepts/services-networking/service/) within the cluster.

The basic idea behind the `Ingress` is to provide a way of describing higher level traffic management constraints, specifically for HTTP. With `Ingress`, we can define rules for routing traffic without creating a bunch of Load Balancers or exposing each service on the node. It can be configured to give services externally-reachable URLs, load balance traffic, terminate SSL/TLS, and offer name-based virtual hosting and content-based routing.

The name Ingress may be a bit misleading, since, at first glance it makes it seem as if it were only for north-south traffic, but actually it can be used for east-west traffic as well.

Lets see a basic example:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: echo-service
  namespace: default
spec:
  rules:
  - host: echo.example.com
    http:
      paths:
      - backend:
          serviceName: echo
          servicePort: 80
        path: /
```

### Ingress controller [🔗︎](#ingress-controller)

Ingress is one of the built-in APIs which doesn’t have a built-in controller, and an ingress controller is needed to actually implement the `Ingress API`.

Ingress is made up of an Ingress API object and an Ingress controller. As mentioned earlier, Kubernetes Ingress is an API object that describes the desired state for exposing services deployed to a Kubernetes cluster. So, to make it work an Ingress controller you will require the actual implementation of the Ingress API to read and process the Ingress resource’s information.

An ingress controller is usually an application that runs as a pod in a Kubernetes cluster and configures a load balancer according to Ingress Resources. The load balancer can be a software load balancer running in the cluster or a hardware or cloud load balancer running externally. Different load balancers require different ingress controllers.

Since the Ingress API is actually *just* metadata, the Ingress controller does the heavy lifting. Various ingress controllers [are available](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/#additional-controllers) and it is important to choose the right one carefully for each use case.

It’s also possible to have multiple ingress controllers in the same cluster and to set a desired ingress controller for each Ingres. Usually, we end up using a combination of these controllers for different scenarios in the same cluster. For example, we may have one for handling the external traffic coming into the cluster which includes bindings to SSL certificates, and have another internal one with no SSL binding that handles in-cluster traffic.

 [![[Raw/Media/Resources/df5ff058a7ffafd584c0ef8e2f210fd9_MD5.png]]](#ingresspng)[![[Raw/Media/Resources/6a42a5779ec7f293a7cb28fe5888f5cd_MD5.png]]](#_)

### Ingress examples [🔗︎](#ingress-examples)

The Ingress [spec](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status) contains all the information needed to configure a load balancer or a proxy server. Most importantly, it contains a list of rules matched against all incoming requests. Ingress resources only support rules for directing HTTP traffic. Besides the base functionality the resource provides, the various ingress controller implementations usually provide several advanced features through custom resource annotations.

#### Single service ingress [🔗︎](#single-service-ingress)

An Ingress with no rules sends all traffic to a single default backend. If none of the hosts or paths match the HTTP request in the Ingress objects, the traffic is routed to the default backend.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: echo-service
  namespace: default
spec:
  backend:
    serviceName: echo
    servicePort: 80
```

#### Simple fanout [🔗︎](#simple-fanout)

A fanout configuration routes traffic based on the HTTP URI of the request. It allows us to use a single load balancer and IP address to serve multiple backend services.

 [![[Raw/Media/Resources/a70c29fa2980eaae297d14fbbfceb991_MD5.png]]](#ingress-fanoutpng)[![[Raw/Media/Resources/228226f39151075578dbccb884cc1232_MD5.png]]](#_)

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-service
  namespace: default
spec:
  rules:
  - host: services.example.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
        path: /service1
      - backend:
          serviceName: service2
          servicePort: 80
        path: /service2
```

#### Hostname based routing [🔗︎](#hostname-based-routing)

Hostname-based routing supports having one load balancer to handle traffic for different hostnames pointing to the same IP address.

 [![[Raw/Media/Resources/7214015599e43ed5d7f6de519bd06b8f_MD5.png]]](#ingress-host-basedpng)[![[Raw/Media/Resources/a16180e49769b81705989e1e27a04f9d_MD5.png]]](#_)

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: public-services
  namespace: default
spec:
  rules:
  - host: service1.example.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
        path: /
  - host: service2.example.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80
        path: /service2
```

#### TLS [🔗︎](#tls)

Ingress can also provide TLS support, but it is limited to port 443 only. If the TLS configuration section in an Ingress specifies different hosts, they are multiplexed on the same port according to the hostname that’s been specified through the SNI TLS extension (if the Ingress controller supports SNI). The TLS secret must contain keys named `tls.crt` and `tls.key`, which contain the certificate and private key for TLS.

 [![[Raw/Media/Resources/c4151cdd3b23911d0c39f9a1204fbd4c_MD5.png]]](#ingress-tlspng)[![[Raw/Media/Resources/80c99f22c35331f3c7186d1266ef4f00_MD5.png]]](#_)

```
apiVersion: v1
kind: Secret
metadata:
  name: public-services-tls
  namespace: default
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
type: kubernetes.io/tls
```

Referencing this secret in an Ingress tells the Ingress controller to secure the channel from the client to the load balancer using TLS. We need to make sure the TLS secret we have created came from a certificate that contains a Common Name (CN), also known as a Fully Qualified Domain Name (FQDN) for `services.example.com`.

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: services-with-tls
  namespace: default
spec:
  tls:
  - hosts:
      - services.example.com
    secretName: public-services-tls
  rules:
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
        path: /service1
      - backend:
          serviceName: service2
          servicePort: 80
        path: /service2
```

### Multiple ingress controllers [🔗︎](#multiple-ingress-controllers)

As was briefly mentioned previously, it is possible to run multiple ingress controllers within a cluster. Each Ingress should specify a class to indicate which ingress controller should be used, if more than one exists within the cluster.

Before Kubernetes 1.18 an annotation (`kubernetes.io/ingress.class`) was used to specify the ingress class. In 1.18 a new `ingressClassName` field has been added to the Ingress spec that is used to reference the `IngressClass` resource used to implement the Ingress.

```
apiVersion: networking.k8s.io/v1beta1
kind: IngressClass
metadata:
  name: external-lb
spec:
  controller: example.com/ingress-controller
  parameters:
    apiGroup: k8s.example.com/v1alpha
    kind: IngressParameters
    name: external-lb
```

IngressClass resources contain an optional parameters field. This can be used to reference additional configurations for the class in question. We can mark a particular IngressClass as the default for the cluster. Setting the `ingressclass.kubernetes.io/is-default-class` annotation to `true` on an IngressClass resource will ensure that new Ingresses, without an `ingressClassName` field specified, will be assigned to the default IngressClass.

## Ingress the Istio way [🔗︎](#ingress-the-istio-way)

> Looking for an in-depth blogpost about Istio ingress? Read on: [An in-depth intro to Istio Ingress](https://banzaicloud.com/blog/backyards-ingress/).

The `Ingress` resource is relatively easy to use for a wide variety of use cases with simple HTTP traffic, which makes it very popular and commonly used nowadays.

On the other hand this simplicity limits its capabilities as well. The various Ingress controller implementations try to extend the feature set with custom annotations, which can help to get the job done, but they are a really awkward way to extend an API.

Istio for example can also be configured to act as an Ingress API implementation, but that diminishes its broad features, because there is just no way to properly configure a bit more complex traffic routing scenario with Ingress API.

Along with support for `Ingress`, **Istio offers another configuration model**, using the [Istio Gateway](https://istio.io/docs/reference/config/networking/gateway/), [VirtualService](https://istio.io/docs/reference/config/networking/virtual-service/) and [DestinationRule](https://istio.io/docs/reference/config/networking/destination-rule/) resources. This model provides a way better and extensive customisation and flexibility than `Ingress`, and allows Istio features such as monitoring, tracing, authz and route rules to be applied to traffic entering the cluster.

The `Gateway` resource describes the port configuration of the gateway deployment, which operates at the edge of the mesh and receives incoming or outgoing HTTP/TCP connections. The specification describes a set of ports that should be exposed, the type of protocol to use, TLS configuration – if any – of the exposed ports, and so on.

`VirtualService` defines a set of traffic routing rules to apply when a host is addressed. Each routing rule defines matching criteria for the traffic of a specific protocol. If the traffic matches a routing rule it is sent to a named destination service that’s defined in the registry. For example, these rules can route requests to different versions of a service or to a completely different service than the one that was requested. Requests can be routed based on the request source and destination, HTTP paths and header fields, and weights associated with individual service versions.

`DestinationRule` defines policies that apply to traffic intended for a service, after routing has occurred. These rules specify load balancing configurations, connection pool sizes from the sidecar, and outlier detection settings to detect and evict unhealthy hosts from the load balancing pool.

Besides having a more fine-grained traffic routing configuration, this model also supports a better security by providing an option to restrict each of these resources separately, with Kubernetes RBAC.

 [![[Raw/Media/Resources/f931c20c5085d38a37e8ce75eca455c0_MD5.png]]](#ingress-istiopng)[![[Raw/Media/Resources/aa934690af0588fe636f1ebad6878df7_MD5.png]]](#_)

To read more about this approach and in detail, check our other post about [ingress/egress gateways with Istio](https://banzaicloud.com/blog/istio-multiple-gateways/). Looking for an in-depth blogpost about Istio ingress? Read the [An in-depth intro to Istio Ingress](https://banzaicloud.com/blog/backyards-ingress/).

## Future of Ingress, the Service APIs [🔗︎](#future-of-ingress-the-service-apis)

There is **ongoing** work on what is probably going to be the next iteration of ingress, called `Service APIs`. It basically comes from the experience the community has gained with the current Ingress implementation.

When Ingress was first designed the focus was on simplicity and centered around the developer, but nowadays it’s much more of a multi-role environment. One common example of this is when an infrastructure or SRE team provides ingress as a service to application developers.

Service APIs take a similar approach as the Istio model does. It splits up the API into different resources for different purposes in order to provide separation and role-based control of who can create ingress, consume an ingress and also provide support that specifies more complex routing and load balancing features on both L4 and L7 protocols. Service APIs also make the model more extendable for future use-cases.

### Service APIs adds the following new resources to the Kubernetes API [🔗︎](#service-apis-adds-the-following-new-resources-to-the-kubernetes-api)

`GatewayClass` and `Gateway` define the type of LB infrastructure that the user’s Service will be exposed on. This is modelled on existing work in the Kubernetes IngressClass types.

A `Route` describes a way to handle traffic given protocol-level descriptions of the traffic. The route is a pure configuration and can be referenced from multiple gateways. There are different route resources for each protocol, e.g. `HTTPRoute`, `TLSRoute`.

The combination of `GatewayClass`, `Gateway`, `*Route` and `Service`(s) will define a load-balancer configuration. The diagram below illustrates the relationships between the different resources:

 [![[Raw/Media/Resources/6103ed4bff61ff127b7f0508a9448327_MD5.png]]](#service-apispng)[![[Raw/Media/Resources/17dc7cbdf3e74aeafba803910d037e7b_MD5.png]]](#_)

These new APIs are not intended to replace any existing APIs, but instead provide a more configurable alternative for complex use cases. To read more about the current state of `Service APIs` check Kubernetes [github page](https://github.com/kubernetes-sigs/service-apis) or its [docs](https://github.com/kubernetes-sigs/service-apis).

## The Future of Ingress [🔗︎](#the-future-of-ingress)

The Ingress API is on its way toward graduating from beta to a stable API in Kubernetes 1.19. After that, it’ll most probably go into maintenance mode and will continue to provide a simple way of managing inbound network traffic for Kubernetes workloads. We should note that there are other approaches to ingress on Kubernetes, and that work is also currently underway on a new, highly configurable set of APIs that will provide an alternative to Ingress in the future.