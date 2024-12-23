---
title: "Understanding Kubernetes Ingress: Deep dive in the real load balancer + Minikube Demo"
source: https://medium.com/@kh.hajed/understanding-kubernetes-ingress-deep-dive-in-the-real-load-balancer-minikube-demo-ac91f2e8ab02
clipped: 2024-12-23
published: 
category: k8s
tags:
  - network
read: false
---

![[Raw/Media/Resources/91407ca92826a3ade50ff4a79b7035b7_MD5.png]]

Exposing applications outside of Kubernetes is one of the most important goals in the deployment process—it's ultimately the end goal. For this reason, we seek resources that can expose and manage traffic from pods to external users, commonly referred to as Edge resources.

Indeed, it is a load balancer, but which one? The Service Load Balancer or the Ingress resource?

In this article, we will explore the differences between Kubernetes Services (ClusterIP, NodePort, ExternalName, and LoadBalancer) and Ingress load balancers. We'll provide a historical review, delve into possible implementations and use cases, and finally, present a simple demo on Minikube.

Let’s begin with a bit of history to see how things were in the early days. When the first release version of Kubernetes 1.0.0 came out in 2015, the only solution to expose traffic was through one of the Kubernetes Services.

Primarily, there were four main service types:

-   **ClusterIP:** Exposes the service on an internal IP in the cluster **only**, allowing other services within the cluster to communicate with it.
-   **NodePort**: Exposes the service on a static port on each node’s IP, enabling access from outside the cluster.
-   **LoadBalancer**: Provisions an external load balancer in the cloud provider’s infrastructure to distribute traffic to the service.
-   **ExternalName**: Maps the service to the contents of the `**externalName**` field, enabling the service to be accessed by an external name.

Simply put, by process of elimination, we can discard ClusterIP, which is an internal-only service. NodePort relies on NodeIPs, which are ephemeral and external, and ExternalName uses CNames instead of labels, which doesn’t suit our case. Therefore, the LoadBalancer emerges as the best solution for our needs.

Here is an example of a LoadBalancer service manifest :

apiVersion: v1  
kind: Service  
metadata:  
  name: example-service  
spec:  
  selector:  
    app: example  
  ports:  
    \- port: 8765  
      targetPort: 9376  
  type: LoadBalancer

After applying this service manifest in a managed Kubernetes environment and waiting for a few seconds to allow Kubernetes to work its magic behind the scenes, you will end up with something similar to this:

NAME            TYPE                 CLUSTER-IP      EXTERNAL-IP       PORT(S)        AGE  
my-service   LoadBalancer           10.100.200.20   203.0.113.10     80:32452/TCP     3h

At this point, the application becomes accessible over the external IP address 10.100.200.20, thanks to the Cloud Controller Manager (CCM). This component in Kubernetes managed clusters acts as a bridge between Kubernetes resources and cloud services. Behind the scenes, when CCM detects the creation of a new LoadBalancer service, it interacts with the cloud provider’s infrastructure to create an actual load balancer. This load balancer then maps the traffic from the service inside the cluster to the load balancer in the cloud.

Now, imagine if we have a microservices architecture. Here’s how the final result would look:

![[Raw/Media/Resources/c7cb5b397515bd59fee23fc94b730b69_MD5.png]]

This architecture can become costly because it requires a dedicated load balancer for each service. Therefore, the need for a centralized solution that handles all traffic at a single point arises. This is where Ingress comes into play. The Ingress resource was added to Kubernetes in December 2015, starting from version 1.1.0 (as a beta resource). Here is how the best practice implemented by ingress works :

![[Raw/Media/Resources/d6da4a42611cc3d819046092d125b506_MD5.png]]

Ingress is a Kubernetes edge resource designed to expose service traffic to a public URL. It offers advanced load balancing utilities such as request interception, SSL/TLS termination, name-based virtual hosting, routing rules (host, path, etc.), and much more.

Those routing features are configured based on declarative rules in the ingress manifest, refer to [Kubernetes Ingress documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/) for more details, here is an example of the routing options:

apiVersion: networking.k8s.io/v1  
kind: Ingress  
metadata:  
  name: ingress-wildcard-host  
spec:  
  rules:  
  \- host: "foo.bar.com"   
    http:  
      paths:  
      \- pathType: Prefix    
        path: "/bar"   
        backend:  
          service:     
            name: service1  
            port:  
              number: 80

In the new architecture, the load balancer is managed by Ingress, specifically implemented to handle Ingress resources. This is one of the reasons why there’s a need for a resource to manage Ingress, known as an Ingress controller. Depending on the environment architecture, at least one of the Ingress controllers maintained by the Kubernetes community must be deployed in the cluster.

There are two main types of Ingress controllers, each with completely different implementation models.

-   **Ingress based on a web server**
-   **Ingress based on a cloud load balancer**

Lets dive deeper and study the Pros and cons of each one ;)

This type of Ingress controller is based on an actual web server such as NGINX or HAproxy. Behind the scenes, when this type of controller is deployed, it starts an NGINX web server in the cluster. The controller’s job is to convert the Ingress configuration template provided in the Ingress manifest to an actual configuration for the web server and update the generic configuration file.

For instance, if we are using the Ingress-Nginx controller, every time an Ingress is deployed, an update is made to the nginx.conf file on the controller side to include the newly created endpoint.

For example, consider this Ingress manifest:

apiVersion: networking.k8s.io/v1  
kind: Ingress  
metadata:  
  name: example-ingress  
  annotations:  
    nginx.ingress.kubernetes.io/rewrite-target: /  
spec:  
  rules:  
  \- host: example.com  
    http:  
      paths:  
      \- path: /app  
        pathType: Prefix  
        backend:  
          service:  
            name: app-service  
            port:  
              number: 80

The equivalent configuration in Nginx is a s below:

http {  
  server {  
    server\_name example.com;

    location /app {  
      proxy\_pass http://app-service:80;  
    }  
  }  
}

This approach has various advantages and disadvantages. In summary here is how it works under the hood and here are some of them:

![[Raw/Media/Resources/f6093693376f644b028b32167ca4105a_MD5.png]]

As shown in the figure above, for this type of Ingress controller we need a layer 4 (transport layer) load-balancer and the Ingresss web-server , in this example Nginx, will handle the application network traffic.

SSL termination is set up exclusively outside the Kubernetes cluster, specifically preceding the Ingress resource. This setup allows for an easy SSL/TLS certificate to be configured since there is only one point of ingress traffic which is part of the cluster itself.

## Pros :

-   Kubernetes Native solution and centrelized configuration.
-   TLS management, We can deploy Certificate management tools to manage certificate lifeCycle and advanced security feature in kubernetes cluster itself.
-   Monitoring, we can expose and handle in cluster level diffrent request signals (Latency, traffic, errors…)

## Cons:

-   Resource Consumption: while it is been deployed to kubernetes we need to consider a high resource consumption for load balancing , routing rules execution and other configuration.
-   **Maintenance Overhead**: It is not a manged service so we need to manage upgrades and support by ourselves.

On the other hand, Cloud Load Balancer Ingress controllers depend on cloud-managed services. Essentially, all load balancing processes are managed by provisioned load balancers in the cloud, and the traffic is directed by the Cloud Controller Manager to the targets. In this implementation mode, we operate outside of the Kubernetes context, as all configurations are made in a different type of load balancer external to the cluster.

![[Raw/Media/Resources/20ad6572b3690fc9d128970eec32f390_MD5.png]]

Let’s consider a concrete example from one of the most widely used cloud providers, AWS. AWS has implemented an Ingress controller based on AWS Application Load Balancer (ALB), known as “aws-alb-ingress-controller”:

![[Raw/Media/Resources/76fd22b3fd9612c00f1f35fb03c040aa_MD5.jpg]]

For further info check Amazon official documentation, [Kubernetes Ingress with AWS ALB Ingress Controller](https://aws.amazon.com/blogs/opensource/kubernetes-ingress-aws-alb-ingress-controller/).

## Pros

-   Native integration in managed clusters (EKS, AKS, GKE …).
-   Managed Service: No Maintenance and less responsibility.
-   High availability: while it uses AWS ALB so it is autoscaling.

## Cons

-   Cost: managed Services are pricy.
-   Limited accessibility and needs cloud provider knowledge.

Enough talking let’s get our hands dirty with this stuff !  
For the demo we are going to use minikube single node cluster. Check the [official guide](https://minikube.sigs.k8s.io/docs/start/) for installation. First verify that your CRI is running ( Docker descktop in my case ) and launch minikube :

hajed-kh@Hajeds-MacBook-Pro ~ % minikube start  
😄  minikube v1.32.0 on Darwin 13.1 (arm64)  
✨  Using the docker driver based on existing profile  
👍  Starting control plane node minikube in cluster minikube  
🚜  Pulling base image ...  
🔄  Restarting existing docker container for "minikube" ...  
🐳  Preparing Kubernetes v1.28.3 on Docker 24.0.7 ...  
🔗  Configuring bridge CNI (Container Networking Interface) ...  
🔎  Verifying Kubernetes components...  
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default

Now install ingress-controller by enabling it in minikube as follow:

hajed\-kh@Hajeds\-MacBook\-Pro ~ % minikube addons enable ingress  
💡  ingress is an addon maintained by Kubernetes. For any concerns contact minikube on GitHub.  
You can view the list of minikube maintainers at: https://github.com/kubernetes/minikube/blob/master/OWNERS  
💡  After the addon is enabled, please run "minikube tunnel" and your ingress resources would be available at "127.0.0.1"  
    ▪ Using image registry.k8s.io/ingress\-nginx/controller:v1.9.4  
    ▪ Using image registry.k8s.io/ingress\-nginx/kube\-webhook\-certgen:v20231011\-8b53cabe0  
    ▪ Using image registry.k8s.io/ingress\-nginx/kube\-webhook\-certgen:v20231011\-8b53cabe0  
🔎  Verifying ingress addon...  
🌟  The 'ingress' addon is enabled  

Let’s check the newly created namespace ingress-nginx :

hajed-kh@Hajeds-MacBook-Pro ~ % kubectl get pods -n ingress-nginx  
NAME                                        READY   STATUS      RESTARTS      AGE  
ingress-nginx-admission-create\-nmg7m        0/1     Completed   0             2min  
ingress-nginx-admission-patch-wvzd5         0/1     Completed   0             2min  
ingress-nginx-controller\-7c6974c4d8-rd4mh   1/1     Running     1             2min

Now let’s deploy and expose a hello world application :

hajed\-kh@Hajeds\-MacBook\-Pro ~ % kubectl create deployment web   
deployment.apps/web created  
hajed\-kh@Hajeds\-MacBook\-Pro ~ % kubectl expose deployment web    
service/web exposed

Now let’s create an ingress with single rule to publish the application in public :

apiVersion: networking.k8s.io/v1  
kind: Ingress  
metadata:  
  name: example-ingress  
  annotations:  
    nginx.ingress.kubernetes.io/rewrite-target: /  
spec:  
  rules:  
    \- host: hello-world.info  
      http:  
        paths:  
          \- path: /  
            pathType: Prefix  
            backend:  
              service:  
                name: web  
                port:  
                  number: 8080

wait till the ingress is read and has an ddress assigned to it :

hajed\-kh@Hajeds\-MacBook\-Pro ~ % kubectl get ingress  
NAME              CLASS   HOSTS              ADDRESS        PORTS   AGE  
example\-ingress   nginx   hello\-world.info   192.168.49.2   80      5m

check now the ingress nginx controller configuration file :sing the command below:

kubectl exec -it ingress-nginx-controller-7c6974c4d8-rd4mh  -n ingress-nginx -- cat /etc/nginx/nginx.conf

You will find a new server configuration added as below for the hello-world ingress :

   server {  
  server\_name hello-world.info ;

    listen 80  ;  
  listen 443  ssl http2 ;

    set $proxy\_upstream\_name "-";  
\--  
   proxy\_redirect                          off;

     }

   } 

And finally Let’s try to access our hello-world app :

![[Raw/Media/Resources/8939166ff68d56e2dcd107387682bee6_MD5.png]]

***Euuuuuheuuu ! it is working*** 🎉🎉🎉

In a Kubernetes cluster, Ingress serves as the primary load balancer, and the choice of implementation depends on your specific requirements. This article aims to demystify Ingress by delving into its underlying mechanisms and highlighting the distinctions between web-based and cloud-based ingress controllers.