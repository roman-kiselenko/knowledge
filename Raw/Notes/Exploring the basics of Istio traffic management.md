---
title: Exploring the basics of Istio traffic management
source: https://medium.com/@arivermar/exploring-the-basics-of-istio-traffic-management-cee13f0817c2
clipped: 2025-01-10
published: 
category: k8s
tags:
  - network
read: false
---

In microservices architecture, the concept of a service mesh has already become firmly established. A service mesh could be defined as an infrastructure layer dedicated to managing and controlling communication between microservices. It provides capabilities such as security, scalability, high availability, observability and more.

There are several solutions that implement the concept of a service mesh, but today we will talk about Istio.

In this series of articles, we will delve into the capabilities that Istio offers us. We will start by analyzing traffic management, then move on to security and observability.

All the scenarios we will present pertain to the implementation of Istio in a Kubernetes cluster, specifically in Minikube.

Let’s review the architecture proposed by Istio:

-   **Control plane:** It configures the proxies to route traffic and is also responsible for managing certificates. In Kubernetes, it is instantiated as a pod called `istiod`. It is composed of three components: Pilot, Citadel, and Galley. Pilot is responsible for abstracting the specific configuration so that any sidecar following the standard can consume it. Citadel handles authentication(certs), and Galley validates the configuration and provides it to all components.
-   **Data plane:** It consists of a set of proxies (Envoy) that are deployed as sidecars in the microservices’ pods and control traffic. Additionally, they collect traffic metrics within the mesh.

![[Raw/Media/Resources/249f1328e71a13329d39e9071dda0f81_MD5.png]]

## Istio installation

There are several methods to install Istio, such as using a Helm Chart, manifests (I don’t recommend you) or the `istioctl` binary (prefered). In our case, Minikube has the Istio addon, so we simply [activate it](https://minikube.sigs.k8s.io/docs/handbook/addons/istio/):

minikube addons enable istio-provisioner  
minikube addons enable istio

With this commands, we will first install the Istio CRDs, followed by `istiod` and `istio-ingressgateway` in the `istio-system` namespace.

![[Raw/Media/Resources/48940d96d06e9822982a46f91ce1cc0e_MD5.png]]

We previously discussed what `istiod` is, so we won’t repeat that, but there are other concepts I’d like to explain. To begin with, `istio-ingressgateway` is the ingress created by Istio and **all incoming traffic to the cluster should go through it**. The Istio ingress gateway is exposed outside the cluster through a LoadBalancer service (in our specific case). In other environments, there might be variations in how this ingress is exposed, but the key point to remember is that all incoming traffic to the mesh must pass through this gateway.

To expose our ingress gateway outside the cluster, we need to run the following command: `minikube tunnel`. This provides an external IP to the LoadBalancer service so that it can be accessed from our local machine

![[Raw/Media/Resources/7974a2f957b8eecb4d5248addfa628cf_MD5.png]]

## Deploying our first microservice

Let’s deploy our first microservice, a simple “hello world.” We will instantiate this service in a new namespace called `hello-world`. The first thing we need to do is label the new namespace with the label **istio-injection=enabled** so that the Envoy proxy is automatically deployed with the service.

kubectl label namespace hello-world istio-injection=enabled

Once the microservice is deployed, we should see 2 contianers:

![[Raw/Media/Resources/bd7c63cd9f43c45184485f78b2147c60_MD5.png]]

Performing a describe of the pods we should see the sidecar:

![[Raw/Media/Resources/61245df72a5ab3fca747da30eada8a8a_MD5.png]]

The first Istio resource we will deploy is called a `DestinationRule`. This object is responsible for defining policies that will be applied to a certain Envoy proxy, and it is important to know that all actions defined in DestinationRules take place after routing to the pod. Policies can specify rules for traffic balancing between endpoints (pods), circuit breaking, or differentiating between subsets of the service.

Let’s try to instantiate a DestinationRule that balances the traffic between the pods with a RANDOM policy:

apiVersion: networking.istio.io/v1alpha3  
kind: DestinationRule  
metadata:  
  name: hello-world-dr  
  namespace: hello-world  
spec:  
  host: hello-world  
  trafficPolicy:  
    loadBalancer:  
      simple: RANDOM

Note: With the `spec.host` field, we indicate which service in the mesh the `DestinationRule` configuration applies to.

This block can be very interesting because we could define a `LEAST_REQUEST` policy, which favors endpoints with fewer requests, thus balancing the traffic between our pods more efficiently.

To expose our service outside the cluster, we create a `VirtualService` and a `Gateway`. Although we won't focus on them for now (let's first dive deeper into `DestinationRules`), in broad terms, the `Gateway` is a rule that applies to Istio's ingress to listen on specific ports for certain hosts. Then, the `VirtualService` define rules for the proxy so that it listens to the requests received at the ingress and routes them to the application service.

apiVersion: networking.istio.io/v1alpha3  
kind: Gateway  
metadata:  
  name: hello-world-gateway  
  namespace: hello-world  
spec:  
  selector:  
    istio: ingressgateway  
  servers:  
  \- name: hello-world  
    port:  
      number: 8080  
      name: http  
      protocol: HTTP  
  hosts:  
  \- "\*"

apiVersion: networking.istio.io/v1alpha3  
kind: VirtualService  
metadata:  
  name: hello-world-vs  
  namespace: hello-world  
spec:  
  hosts:  
  \- test.example.com  
  gateways:  
  \- hello-world-gateway  
  http:  
  \- route:  
    \- destination:  
        host: hello-world

DestinationRules also permit to add sesison affinity policies which are useful when users should mantain an open session against the same pod.

apiVersion: networking.istio.io/v1   
kind: DestinationRule   
metadata:   
  name: hellow-world-dr   
  namespace: hello-world  
spec:   
  host: hello-world  
  trafficPolicy:   
    loadBalancer:   
      consistentHash:   
        httpHeaderName: x-forwarded-for

If we execute multiple curls to the url test.example.com (I’ve created a new entry in my /etc/hosts file so this url is pointing to Istio ingress LB IP) we should see that all the times the same pod is answering:

count = 0  
while count < 10 :  
  curl http://test.example.com  
  count +=1

Answer: hello-world-5756999584-zwdgb

## Circuit breakers

One of the biggest advantages Istio offers is that it enforces circuit breaking limits at the network level. Circuit breaking is characterized by preventing a failure from recurring constantly and protecting the application during load spikes. Also make requests fail fast so host is released from more petitions that could overload it.

One of the features introduced by `DestinationRules` to implement circuit breaking is the eviction of pods from the load-balancing pool if they experience recurring errors. This is very useful because if one of the endpoints ends up in an inconsistent state or is restarted for some reason, it would be temporarily removed from the load balancing, only to be added back after a while.

apiVersion: networking.istio.io/v1   
kind: DestinationRule   
metadata:   
  name: hello-world-dr  
spec:   
  host: hello-world  
    outlierDetection:   
      consecutive5xxErrors: 7   
      interval: 5m   
      baseEjectionTime: 15m

It is only needed to specify the number of 5xx errors (*consecutive5xxErrors*) that must occur consecutively for the pod to be removed from the pool, and the time interval for reevaluating the pod’s response status (*interval*). The *baseEjectionTime* is also defined, which is the minimum time during which the pod will remain out of the pool. Each time the pod’s status is reevaluated, if it responds with a 5xx error again, this exclusion time is increased as follows: *exclusion = n x baseEjectionTime*, where n is the number of times the pod has been excluded from the pool.

Another implementation of a Circuit Breaker is through the configuration of the connectivity pool. In this case, traffic is distinguished between HTTP and TCP. If we focus on HTTP, we can implement the following `DestinationRule`:

apiVersion: networking.istio.io/v1alpha3  
kind: DestinationRule  
metadata:  
  name: hello-world-dr  
  namespace: hello-world  
spec:  
  host: hello-world  
  trafficPolicy:  
    connectionPool:  
      http:  
        http1MaxPendingRequests: 1  
        maxRequestsPerConnection: 1

With this manifest, we specify that the maximum http1 number of pending requests is 1, and the maximum number of connections per request is also 1. It should be noted that we are always talking about requests per host (pod).

To test this, we can use a tool called Fortio, which is used for load testing. We simply start a container in our local network with the Fortio image and run a command to create multiple concurrent connections to our service:

docker run --network host fortio/fortio load -c 4 -qps 0 -n 20 -loglevel Warning   
  -X POST http://test.example.com

Code 200 : 5 (25.0 %)  
Code 503 : 15 (75.0 %)  
Response Header Sizes : count 20 avg 63 +/- 109.1 min 0 max 252 sum 1260  
Response Body/Total Sizes : count 20 avg 420.5 +/- 300.5 min 247 max 941 sum 8410  
All done 20 calls (plus 0 warmup) 2.461 ms avg, 1250.3 qps

In this way, we can see that the circuit breaker has cut off some requests. If we repeat this test several times, we will notice that the proxy has some “leeway” and does not always cut connections when the maximum number is reached, so the result will not always be the same.

We can also limit connections at the TCP level as follows:

apiVersion: networking.istio.io/v1alpha3  
kind: DestinationRule  
metadata:  
  name: hello-world-dr  
  namespace: hello-world  
spec:  
  host: hello-world  
  trafficPolicy:  
    connectionPool:  
      tcp:  
        maxConnections: 2

## Subsets, Deploying Multiple Versions of Our Service

Now we want to roll out a new version of our microservice, here is where subsets come into play. These allow us to perform A/B testing or direct traffic to specific versions of the service. **Policies can also be defined at the subset level, and these override the policies at the service level.**

We deploy a new version of our application with a slight modification, so now we have 2 different versions deployed under the same service. The deployment for the first version has the label `version: v1`, and the second version has the label `version: v2`. Thus, we define 2 subsets based on the labels of each deployment.

Next, we modify the `DestinationRule`:

apiVersion: networking.istio.io/v1alpha3  
kind: DestinationRule  
metadata:  
  name: hello-world-dr  
  namespace: hello-world  
spec:  
  host: hello-world  
  subsets:  
  \- name: v1  
    labels:  
      version: v1  
  \- name: v2  
    labels:  
      version: v2

Along with our `DestinationRule`, we deploy a `VirtualService`. This directs HTTP traffic to the `hello-world` service and to the `v1` subset, except for the path `/v2`, which directs traffic to the `v2` subset (i.e., the v2 of our microservice) and also rewrites the path `/v2` to `/`. The latter is for compatibility with the Apache route.

apiVersion: networking.istio.io/v1alpha3  
kind: VirtualService  
metadata:  
  name: hello-world-vs  
  namespace: hello-world  
spec:  
  hosts:  
  \- test.example.com  
  gateways:  
  \- hello-world-gateway  
  http:  
  \- route:  
    \- destination:  
        host: hello-world  
        subset: v2  
        port:  
          number: 80  
    match:  
    \- uri:  
        prefix: /v2  
    rewrite:  
      uri: /  
  \- route:  
    \- destination:  
        host: hello-world  
        subset: v1  
        port:  
          number: 80

Finally we deploy a Gateway to recieve requests from outside the cluster:

apiVersion: networking.istio.io/v1alpha3  
kind: Gateway  
metadata:  
  name: hello-world-gateway  
  namespace: hello-world  
spec:  
  selector:  
    istio: ingressgateway  
  servers:  
  \- name: hello-world  
    port:  
      number: 80  
      name: http  
      protocol: HTTP  
    hosts:  
    \- test.example.com

If we search in th browser for path /v2 we see the following:

![[Raw/Media/Resources/6bccbfecb4003e2e0b3ea986e1d4205b_MD5.png]]

While if we go to / :

![[Raw/Media/Resources/ab8d88832b353d395819e185f7cef189_MD5.png]]

This feature also permits us to implement a canary rollout strategy for the upgrade of our microservices, combining subsets with VirtualServices.

apiVersion: networking.istio.io/v1alpha3  
kind: VirtualService  
metadata:  
  name: hello-world-vs  
  namespace: hello-world  
spec:  
  hosts:  
  \- test.example.com  
  gateways:  
  \- hello-world-gateway  
  http:  
  \- route:  
    \- destination:  
        host: hello-world  
        subset: v2  
      weight: 80  
  \- route:  
    \- destination:  
        host: hello-world  
        subset: v1  
      weight: 20

In this way, we direct 80% of the traffic to version 1 of our service and the remaining 20% to the new version, limiting the impact of a potential failure in the new version while exposing it to real traffic.

## Conclusions

As you can see, Istio provides a wealth of useful features to improve the reliability and resilience of our systems, and it enables us to apply various rollout methods safely. What has been explained in this article are only the basics, thare are lot more useful features that make networking better, if you want to explore them just read through the Istio Documentation.

## References

Istio Architecture: [https://istio.io/latest/docs/ops/deployment/architecture/](https://istio.io/latest/docs/ops/deployment/architecture/)  
Istio Minikube: [https://minikube.sigs.k8s.io/docs/handbook/addons/istio/](https://minikube.sigs.k8s.io/docs/handbook/addons/istio/)  
DestinationRules: [https://istio.io/latest/docs/reference/config/networking/destination-rule/](https://istio.io/latest/docs/reference/config/networking/destination-rule/)  
Fortio: [https://github.com/fortio/fortio](https://github.com/fortio/fortio)