---
title: How to expose Kubernetes services using Istio API gateway from scratch?
source: https://medium.com/@joeri_verhavert/how-to-expose-kubernetes-services-using-istio-api-gateway-from-scratch-e5a8709afda4
clipped: 2025-01-12
published: 
category: network
tags:
  - network
read: false
---

In Kubernetes, managing external access to services within a cluster can be efficiently handled using an ingress gateway. An ingress gateway allows you to define rules for routing external traffic to various services, simplifying the management of complex traffic patterns. By using a single ingress gateway, you can centralize and streamline the configuration, enhance security measures, and reduce operational costs. This approach is particularly beneficial in environments where multiple services need to be exposed to external clients, providing a unified and consistent entry point into the cluster.

![[Raw/Media/Resources/76fe807a1cbfe661aea086b94f114b74_MD5.png]]

## **Cilium Container Network Interface (CNI)**

We are starting with a Kubernetes cluster that has Cilium installed as the Container Network Interface (CNI). Cilium was chosen for its robust networking capabilities and advanced security features. At this stage, we haven’t enabled an ingress controller nor a Gateway API via Cilium. This allows us to fully focus on implementing Gateway APIs through Istio.

Before diving into the details of configuring and utilising Istio gateways, we will briefly go over the Cilium configuration to provide context and ensure a solid foundation.

Below, you can find the details of the Cilium deployment.

```
hl@istio-master01:~$ kubectl get deploy cilium-operator -n kube-system  
NAME              READY   UP-TO-DATE   AVAILABLE   AGE  
cilium-operator   1/1     1            1           25d
```

  
```
hl@istio-master01:~$ kubectl get pods -n kube-system  
NAME                                     READY   STATUS    RESTARTS      AGE  
cilium-m46q7                             1/1     Running   1 (22d ago)   25d  
cilium-operator-bdf87c665-l77nc          1/1     Running   6 (22d ago)   25d  
cilium-zgln7                             1/1     Running   1 (22d ago)   25d  
cilium-zs65l                             1/1     Running   1 (22d ago)   25d  
cilium-zsxz4                             1/1     Running   1 (22d ago)   25d
```

Depending on your environment, you can either use L2 Announcement or, if your environment is more complex, BGP to advertise routes. Since we’re building this blog case in our homelab, we’ve installed the necessary configurations to use L2 Announcements.

  
```
hl@istio-master01:~$ kubectl get ippool -n kube-system -o yaml  
apiVersion: cilium.io/v2alpha1  
kind: CiliumLoadBalancerIPPool  
metadata:  
  name: cilium-ippool  
  namespace: kube-system  
spec:  
  blocks:  
  - start: 192.168.244.220  
    stop: 192.168.244.225
```

  
```
hl@istio-master01:~$ kubectl get CiliumL2AnnouncementPolicy -o yaml  
apiVersion: cilium.io/v2alpha1  
kind: CiliumL2AnnouncementPolicy  
metadata:  
  name: cilium-l2-policy  
  namespace: kube-system  
spec:  
  nodeSelector:  
    matchExpressions:  
    - key: node-role.kubernetes.io/control-plane  
      operator: DoesNotExist  
  externalIPs: true  
  loadBalancerIPs: true  
  interfaces:  
  - eth0
```

For more details regarding Cilium L2 Announcements: [https://docs.cilium.io/en/latest/network/l2-announcements/](https://docs.cilium.io/en/latest/network/l2-announcements/)

## Istio Service Mesh

Let’s explore how we can use `istioctl` to install Istio. The `istioctl` command-line tool simplifies the installation and management of Istio on your cluster.

By using `istioctl`, you can customize your Istio installation, enable specific features, and ensure that all components are correctly configured. We'll walk through the steps required to install Istio with `istioctl` and highlight key options you can use to tailor the installation to your needs.

  
```
hl@istio-master01:~$ istioctl profile list  
Istio configuration profiles:  
    ambient  
    default  
    demo  
    empty  
    minimal  
    openshift  
    openshift-ambient  
    preview  
    remote  
    stable
```

  
```
hl@istio-master01:~$ istioctl profile dump default  
apiVersion: install.istio.io/v1alpha1  
kind: IstioOperator  
spec:  
  components:  
    base:  
      enabled: true  
    egressGateways:  
    - enabled: false  
      name: istio-egressgateway  
    ingressGateways:  
    - enabled: true  
      name: istio-ingressgateway  
    pilot:  
      enabled: true  
  hub: docker.io/istio  
  profile: default  
  tag: 1.22.1  
  values:  
    defaultRevision: ""  
    gateways:  
      istio-egressgateway: {}  
      istio-ingressgateway: {}  
    global:  
      configValidation: true  
      istioNamespace: istio-system
```

  
```
hl@istio-master01:~$ istioctl install --set profile=default
```

After installing Istio with the default profile, an Ingress Gateway LoadBalancer service is automatically created. This service obtains an external IP from the IP pool that was established during the Cilium installation phase. This IP address of the load balancer serves as the endpoint used to reach the applications externally.

```
hl@istio-master01:~$ kubectl get svc -n istio-system  
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)                                                                      AGE  
istio-ingressgateway   LoadBalancer   10.104.224.57    192.168.244.220   15021:30189/TCP,80:31672/TCP,443:31402/TCP,31400:31758/TCP,15443:31900/TCP   25d  
istiod                 ClusterIP      10.108.197.62    <none>            15010/TCP,15012/TCP,443/TCP,15014/TCP                                        25d
```

## Expose bookInfo application with Istio API gateway’s

Now, let’s proceed to configure a application which we can later on expose through an Istio gateway. We will use the “famous” bookInfo application for the demo.

To enable Istio injection, you need to label the namespace so that Istio knows to add its sidecar containers to each pod within that namespace.

> FYI: Istio injection is not a hard requirement to expose an application via Istio Gateway APIs.

```
hl@istio-master01:~$ kubectl label namespace default istio-injection=enabled
```

Deploy the bookInfo application:

```
hl@istio-master01:~$ kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/bookinfo/platform/kube/bookinfo.yaml
```

Before proceeding with further steps, it’s crucial to confirm that all services and pods in your environment are accurately defined and actively running. This ensures a stable foundation for the next configurations and deployments.

```
hl@istio-master01:~$ kubectl get services  
NAME          TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE  
details       ClusterIP   10.0.0.31    <none>        9080/TCP   6m  
kubernetes    ClusterIP   10.0.0.1     <none>        443/TCP    7d  
productpage   ClusterIP   10.0.0.120   <none>        9080/TCP   6m  
ratings       ClusterIP   10.0.0.15    <none>        9080/TCP   6m  
reviews       ClusterIP   10.0.0.170   <none>        9080/TCP   6m
```

```
hl@istio-master01:~$ kubectl get pods  
NAME                             READY     STATUS    RESTARTS   AGE  
details-v1-1520924117-48z17      2/2       Running   0          6m  
productpage-v1-560495357-jk1lz   2/2       Running   0          6m  
ratings-v1-734492171-rnr5l       2/2       Running   0          6m  
reviews-v1-874083890-f0qf0       2/2       Running   0          6m  
reviews-v2-1343845940-b34q5      2/2       Running   0          6m  
reviews-v3-1813607990-8ch52      2/2       Running   0          6m
```

Finally, verify that the Bookinfo application is running by sending a request to it using a curl command from any pod.

```
hl@istio-master01:~$ kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items\[0\].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.\*</title>"
```

  
<title>Simple Bookstore App</title>

Next, apply the Gateway and VirtualService resources to expose the application. Note that the selector on the Gateway resource targets the Istio Ingress Gateway, referencing a label on the load balancer service of the Istio Ingress Gateway.

To determine the label applied to the Istio Ingress Gateway’s load balancer service, use the following command:

```
hl@istio-master01:~$ kubectl get svc istio-ingressgateway -n istio-system --show-labels  
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)                                                                      AGE   LABELS  
istio-ingressgateway   LoadBalancer   10.104.224.57    192.168.244.220   15021:30189/TCP,80:31672/TCP,443:31402/TCP,31400:31758/TCP,15443:31900/TCP   25d   app=istio-ingressgateway,install.operator.istio.io/owning-resource-namespace=istio-system,install.operator.istio.io/owning-resource=installed-state,istio.io/rev=default,istio=ingressgateway,operator.istio.io/component=IngressGateways,operator.istio.io/managed=Reconcile,operator.istio.io/version=1.22.1,release=istio
```

  
```
hl@istio-master01:~$ kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/bookinfo/networking/bookinfo-gateway.yaml
```

> FYI: You may need to update the port on the Bookinfo Gateway to listen on port 80, as it is configured to use port 8080 by default.

  
```
hl@istio-master01:~$ kubectl edit gateway bookinfo-gateway
```

Confirm that the Bookinfo application is accessible from outside the cluster, run the following `curl` command:

  
```
hl@istio-master01:~$ export INGRESS\_NAME=istio-ingressgateway  
hl@istio-master01:~$ export INGRESS\_NS=istio-system  
hl@istio-master01:~$ export INGRESS\_HOST=$(kubectl -n "$INGRESS\_NS" get service "$INGRESS\_NAME" -o jsonpath='{.status.loadBalancer.ingress\[0\].ip}')  
hl@istio-master01:~$ export INGRESS\_PORT=$(kubectl -n "$INGRESS\_NS" get service "$INGRESS\_NAME" -o jsonpath='{.spec.ports\[?(@.name=="http2")\].port}')  
hl@istio-master01:~$ export GATEWAY\_URL=$INGRESS\_HOST:$INGRESS\_PORT
```

  
```
hl@istio-master01:~$ curl -s "http://${GATEWAY\_URL}/productpage" | grep -o "<title>.\*</title>"
```

  
<title>Simple Bookstore App</title>

And that is how you set up a gateway with Istio. But wait, there’s more to consider! What about using Fully Qualified Domain Names (FQDNs) for your services? Additionally, how do you handle scenarios where multiple services need to be exposed through the same gateway?

## Expose multiple services with Istio API gateway

First, let’s update the host wildcard in the Bookinfo Gateway so that it doesn’t listen to all traffic, but only to a specific FQDN. The great thing about the Istio Gateway is that it functions similarly to virtual hosts in HTTP. By specifying an FQDN in the Gateway resource, you can restrict it to listen only on that FQDN.

This ensures that the Gateway will only handle traffic directed to that particular domain, effectively forwarding it to the correct service. This approach provides better control and security over your exposed services.

```
hl@istio-master01:~$ kubectl edit gateway bookinfo-gateway  
```
  
  
  
  
```
apiVersion: networking.istio.io/v1  
kind: Gateway  
metadata:  
  annotations:  
    kubectl.kubernetes.io/last-applied-configuration: |  
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"Gateway","metadata":{"annotations":{},"name":"bookinfo-gateway","namespace":"default"},"spec":{"selector":{"istio":"ingressgateway"},"servers":\[{"hosts":\["\*"\],"port":{"name":"http","number":8080,"protocol":"HTTP"}}\]}}  
  creationTimestamp: "2024-07-09T11:43:52Z"  
  generation: 2  
  name: bookinfo-gateway  
  namespace: default  
  resourceVersion: "7225679"  
  uid: 4179f01c-6f4e-42d3-8891-7fc8f03f8e6e  
spec:  
  selector:  
    istio: ingressgateway  
  servers:  
  - hosts:  
    - 'bookinfo.example.com'   
    port:  
      name: http  
      number: 80  
      protocol: HTTP
```

Next, update the `/etc/hosts` file to bypass the need for updating records or adding records within the DNS server. This allows to to resolve a FQDN locally. Add the following entry:

```
hl@istio-master01:~$ sudo vi /etc/hosts  
\# Add   
192.168.244.220 bookinfo.example.com
```

```
\# Test FQDN   
hl@istio-master01:~$ ping bookinfo.example.com  
PING bookinfo.example.com (192.168.244.220) 56(84) bytes of data.  
From 192.168.244.203 (192.168.244.203) icmp\_seq=2 Redirect Host(New nexthop: bookinfo.example.com (192.168.244.220))  
From 192.168.244.203 (192.168.244.203) icmp\_seq=3 Redirect Host(New nexthop: bookinfo.example.com (192.168.244.220))  
From 192.168.244.203 (192.168.244.203) icmp\_seq=4 Redirect Host(New nexthop: bookinfo.example.com (192.168.244.220))
```

Since you’ve updated the Bookinfo Gateway to listen on the FQDN, let’s confirm it’s working by using the following curl command:

```
hl@istio-master01:~$ curl -s "http://bookinfo.example.com/productpage" | grep -o "<title>.\*</title>"
```

  
<title>Simple Bookstore App</title>

Let’s us continue by adding a second deployment. We’ll use an Nginx deployment as an example and replicate the entire setup process that we previously performed for the Bookinfo application. This will include configuring the Nginx deployment, creating the necessary service, updating the Istio Gateway to include the new FQDN, and ensuring proper routing with the VirtualService.

By following these steps, we’ll ensure that both applications are correctly exposed and accessible via their respective FQDNs.

```
  
hl@istio\-master01:~$ kubectl create deploy nginx --image=nginx  
deployment.apps/nginx create
```

  
```
hl@istio\-master01:~$ kubectl expose deploy nginx --port=80  
service/nginx expose
```

Apply the following resource to the cluster.

```
\---  
apiVersion: networking.istio.io/v1alpha3  
kind: Gateway  
metadata:  
  name: nginx-gateway  
  namespace: default  
spec:  
  selector:  
    app: istio-ingressgateway  
  servers:  
  \- hosts:  
    \- "nginx.example.com"  
    port:  
      name: http  
      number: 80  
      protocol: HTTP  
\---  
apiVersion: networking.istio.io/v1alpha3  
kind: VirtualService  
metadata:  
  name: nginx  
  namespace: default  
spec:  
  hosts:  
  \- "nginx.example.com"  
  gateways:  
  \- nginx-gateway  
  http:  
  \- route:  
    \- destination:  
        host: "nginx"  
        port:  
          number: 80  
```

After creating the Nginx deployment and adding the Nginx Gateway and VirtualService to listen on the correct FQDN and point to the appropriate ClusterIP service, update your `/etc/hosts` file once more to bypass using a DNS server and resolve your FQDN locally.

```
hl@istio\-master01:~$ sudo vi /etc/hosts  
192.168.244.220 bookinfo.example.com  
\# Add  
192.168.244.220 nginx.example.com

hl@istio\-master01:~$ ping nginx.example.com  
PING nginx.example.com (192.168.244.220) 56(84) bytes of data.  
From 192.168.244.203 (192.168.244.203) icmp\_seq\=2 Redirect Host(New nexthop: bookinfo.example.com (192.168.244.220))  
From 192.168.244.203 (192.168.244.203) icmp\_seq\=3 Redirect Host(New nexthop: bookinfo.example.com (192.168.244.220))  
From 192.168.244.203 (192.168.244.203) icmp\_seq\=4 Redirect Host(New nexthop: bookinfo.example.com (192.168.244.220))
```

Now that everything is set up, let’s test if the gateway is functioning correctly using a curl command:

```
hl@istio-master01:~$ curl nginx.example.com  
<!DOCTYPE html\>  
<html\>  
<head\>  
<title\>Welcome to nginx!</title\>  
<style\>  
html { color-scheme: light dark; }  
body { width: 35em; margin: 0 auto;  
font-family: Tahoma, Verdana, Arial, sans-serif; }  
</style\>  
</head\>  
<body\>  
<h1\>Welcome to nginx!</h1\>  
<p\>If you see this page, the nginx web server is successfully installed and  
working. Further configuration is required.</p\>

<p\>For online documentation and support please refer to  
<a href\="http://nginx.org/"\>nginx.org</a\>.<br/>  
Commercial support is available at  
<a href\="http://nginx.com/"\>nginx.com</a\>.</p\>

<p\><em\>Thank you for using nginx.</em\></p\>  
</body\>  
</html\>
```

Great, it works! This demonstrates how you can effectively utilize API gateways to expose multiple services within your Kubernetes cluster. By configuring Istio Gateways with specific FQDNs and routing rules using VirtualServices, you can securely and efficiently expose different applications to external traffic, ensuring seamless access and management.

## Conclusion

We have configured Istio Gateways to expose multiple services using specific FQDNs and routing rules through VirtualServices. This setup provides a straightforward and effective method for securely managing external traffic to our applications within Kubernetes. Compared to traditional Ingress controllers, Istio Gateways offer enhanced flexibility and control over traffic routing and security policies, making them a preferred choice for modern microservices architectures.