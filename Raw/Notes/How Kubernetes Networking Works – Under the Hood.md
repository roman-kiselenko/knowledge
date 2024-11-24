---
title: How Kubernetes Networking Works – Under the Hood
source: https://www.suse.com/c/advanced-kubernetes-networking/
clipped: 2023-09-04
published: 
category: network
tags:
  - k8s
  - network
read: true
---

By Tobias Gurtzick

Kubernetes networking is a complex topic, if not even the most complicated topic. This post will give you insight on how kubernetes actually creates networks and also how to setup a network for a kubernetes cluster yourself.

This article doesn’t cover how to setup a kubernetes cluster itself, you could use minikube to quickly spin up a test cluster. All the examples in this post will use a rancher 2.0 cluster, but apply everywhere else too. Even if you are planning to use any of the new public cloud managed kubernetes services such as EKS, AKS, GKE or IBM cloud, you should understand how kubernetes networking works.

For a basic introduction to kubernetes networking, please see this post [How Kubernetes Networking Works – The Basics](https://neuvector.com/network-security/kubernetes-networking/).

### How to Utilize Kubernetes Networking

Many kubernetes (k8s) deployment guides provide instructions for deploying a kubernetes networking CNI as part of the k8s deployment. But if your k8s cluster is already running, and no network is yet deployed, deploying the network is as simple as running their provided config file against k8s (for most networks and basic use cases). For example, to deploy flannel:

```

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

With this k8s from a networking perspective is ready to go. To test everything is working we create 2 pods.

```
cat <<EOF | kubectl create -f -
---
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: pod1
    image: docker.io/centos/tools:latest
    command:
      - /sbin/init
---
apiVersion: v1
kind: Pod
metadata:
  name: pod2
spec:
  containers:
  - name: pod2
    image: docker.io/centos/tools:latest
    command:
      - /sbin/init
EOF
```

This will create two pods, which are already utilizing our driver. Looking at one of the containers we find the network with the ip range 10.42.0.0/24 attached.

```

$: kubectl exec -it pod1 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid_lft forever preferred_lft forever
inet6 ::1/128 scope host
valid_lft forever preferred_lft forever
3: eth0@if15: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
link/ether 2a:70:97:76:a0:48 brd ff:ff:ff:ff:ff:ff link-netnsid 0
inet 10.42.2.33/24 scope global eth0
valid_lft forever preferred_lft forever
inet6 fe80::2870:97ff:fe76:a048/64 scope link tentative dadfailed
valid_lft forever preferred_lft forever
```

A quick ping test from the other pod shows us, that the network is working properly.

```

$: kubectl exec -it pod2 ping 10.42.2.33
PING 10.42.2.33 (10.42.2.33) 56(84) bytes of data.
64 bytes from 10.42.2.33: icmp_seq=1 ttl=62 time=0.818 ms
--- 10.42.2.33 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.818/0.818/0.818/0.000 ms
```

### How Does Kubernetes Networking Compared to Docker Networking?

Kubernetes manages networking through CNI’s on top of docker and just attaches devices to docker. While docker with docker swarm also has its own networking capabilities, such as overlay, macvlan, bridging, etc, the CNI’s provide similar types of functions.

It’s also important to mention that k8s does not use docker0, which is docker’ default bridge, but creates its own bridge named cbr0, which was chosen to differentiate from the docker0 bridge.

### Why Do We Need Overlay Networks?

Overlay networks such as vxlan or ipsec (which you might be familiar with from setting up secure VPNs) encapsulate the packet into another packet. This makes entities addressable which are outside of the scope of another machine. Alternatives to overlay networks include L3 solutions such as macvtap(lan) or even L2 solutions such as ivtap(lan), but these come with limitations or even unwanted side effects.

Any solution on L2 or L3 makes a pod addressable on the network. This means a pod is reachable not just within the docker network, but is directly addressable from outside the docker network. These could be public or private ip addresses.

However, communication on L2 is cumbersome and your experience will vary depending on your network equipment. Some switches need some time to register your MAC Address, before it actually gets reachable to the rest of the network. You could also run into trouble because the neighbor (ARP) table of the other hosts in the system still runs on an outdated cache, and you always need to run with dhcp instead of host-local to avoid conflicting ips between hosts. The mac address and neighbor table problems are the reasons solutions such as ipvlan exist. These do not register new mac addresses but route traffic over the existing one, but these also have their own issues.

The conclusion and my recommendation is that for most users the overlay network is the default solution and should be adequate. However as soon as workloads get more advanced with more specific requirements you will want to consider other solutions such as BGP and direct routing instead of overlay networks.

### How Does Kubernetes Networking Work Under the Hood?

The first thing to understand in kubernetes is that a pod is not actually the equivalent to a container, but is a collection of containers. And all these containers of the same collection share a network stack. Kubernetes manages that by setting up the network itself on the pause container, which you will find for every pod you create. All other pods attach to the network of the pause container which itself does nothing but provide the network. Therefore, it is also possible for one container to talk to a service in a different container, which is in the same definition of the same pod, via localhost.

Apart from that local communication, the communication between pods looks pretty much the same as container to container communication in docker networks.

### Kubernetes Traffic Routing

There are two scenarios which I’ll go into more detail to explain how traffic gets routed between pods.

1\. Routing traffic on the same host:  
There are two scenarios where the traffic does not leave the host.This is either when the service called is running on the same node, or it is in the same container collection within a single pod.  
In the case of calling localhost:80 from container 1 in our first pod and having the service running in the container 2, the traffic will pass the network device and forward the packet to its destination. In this case the route the traffic travels is quite short.

It gets a bit longer when we communicate to a different pod. The traffic will pass over to cbr0, which next will notice that we communicate on the same subnet and therefore directly forwards the traffic to its destination pod, as shown below.

![[Raw/Media/Resources/4a16b601d0f19c716aabd24164234995_MD5.webp]]

2\. Routing traffic across hosts:

This gets a bit more complicated when we leave the node. cbr0 will now pass the traffic to the next node, whose configuration is managed by the CNI. These are basically just routes of the subnets with the destination host as a gateway. The destination host can then just proceed on its own cbr0 and forward the traffic to the destination pod, as shown below.

![[Raw/Media/Resources/60e5e4221264aa36dc762cff2feb1701_MD5.webp]]

### What Exactly is a CNI?

A CNI, which is short for Container Networking Interface, is basically an external module with a well defined interface that can be called by kubernetes to execute actions to provide networking functionality.

You can find the maintained reference plugins which include most of the important ones in the official repo of containernetworking here [https://github.com/containernetworking/plugins](https://github.com/containernetworking/plugins).

A CNI version 3.1 is basically not very complicated. It consists of 3 required functions, ADD, DEL and VERSION, which pretty much should do what they sound like for managing the network. For a more detailed description of what each function is expected to return and gets passed, you can read the spec at [https://github.com/containernetworking/cni/blob/master/SPEC.md](https://github.com/containernetworking/cni/blob/master/SPEC.md).

### The Different CNI’s

To give you a bit of orientation we will look at some of the most popular CNI’s.

#### Flannel

Flannel is a simple network and is the easiest setup option for an overlay network. It’s capabilities include native networking but has limitations when using it on multiple networks. Flannel is for most users, beneath Canal, the default network to choose, it is a simple option to deploy, and even provides some native networking capabilities such as host gateways. Flannel has some limitations including lack of support for network security policies and the capability to have multiple networks.

#### Calico

Calico takes a different approach than flannel. It is technically not an overlay network, but rather a system to configure routing between all systems involved. To accomplish this, Calico leverages the BorderGatewayProtocol (BGP) which is used for the Internet in a process named peering, were every peering party exchanges traffic and participates in the bgp network. The BGP protocol itself propagates routes under its ASN, with the difference that these are private and there isn’t a need to register them with RIPE.

However, for some scenarios also Calico works with an overlay network, in this case IPINIP, which is always used when a node is sitting on a different network, in order to enable the exchange of traffic between those two hosts.

#### Canal

Canal is is based on Flannel but with some Calico components such as felix, the host agent, which allows you to utilize network security policies. These are normally missing in Flannel. So it basically extends Flannel with the addition of security policies.

#### Multus

Multus is a CNI that actually is not a network interface itself. It orchestrates multiple interfaces and without an actual network configured, pods couldn’t communicate with multus alone. So multus is an enabler for multi device and multi subnet networks. The graphic below shows how this works, multus itself basically calls the real CNI instead of the kubelet and communicates back to the kubelet the results.

![[Raw/Media/Resources/324240822e14243eb6214c5c2e99b9c6_MD5.webp]]

#### Kube-Router

Also worth mentioning is kube-router, which like Calico works with BGP and routing instead of an overlay network and like Calico utilizes IPINIP where needed. It also utilizes ipvs for loadbalancing.

### Setting up a Multi-Network k8s Cluster

In the cases when you need to utilize multiple networks, you’ll likely be required to use multus. While multus is quite mature and works without too many issues, you should know that there are currently some limitations.

One of that limitations is that portmapping does not work, which is documented and tracked on the following issue on github ([https://github.com/intel/multus-cni/issues/29](https://github.com/intel/multus-cni/issues/29)). This is going to be fixed in the future, but currently should you be in need of mapping ports, either nodePort configs or hostPort configs, you won’t be able to do that due to the bug referenced.

### Setting Up Multus

The first thing we need to do is to setup multus itself. This is pretty much the config from the multus repositories examples, but with some important adjustments. See the link below to the sample.

The first thing was to adjust the config map. Because we plan to have a default network with Flannel, we define the configuration in the delegates array of the multus config. Some important settings here marked in red are “masterplugin”: true and to define the bridge for the flannel network itself. You’ll see why we need this in the next steps. Other than that there isn’t much else to adjust except adding the mounting definition of the config map, because for some reason this was not done in their example.

Another important thing about this config map is that everything defined in this config map are the default networks that are automatically mounted to the containers without further specification. Also should you want to edit this file, please note you either need to kill and rerun the containers of the daemonset or reboot your node to have the changes take effect.

You can view the sample yaml file [HERE](https://neuvector.com/network-security/advanced-kubernetes-networking/#fancyboxID-1) .

### Setting up the Primary Flannel Overlay Network

For the primary Flannel network, things are pretty much very easy. We can take the example from the multus repository for this and just deploy it. The only adjustments that have been made here are the CNI mount, adjustment of tolerations and some adjustments made for the CNI settings of Flannel. For example adding “forceAddress”:true and removing “hairpinMode”: true.  
This was tested on a cluster that was setup with RKE, but should work on other clusters as well as long as you mount the CNIs from your host correctly, in our case /opt/cni/bin.

The multus team themself did not really change much; they only commented out the initcontainer config, which you could just safely delete. This is made since multus will setup its delegates and will act as the primary “CNI.”

You can view the modified Flannel daemonset [HERE](https://neuvector.com/network-security/advanced-kubernetes-networking/#fancyboxID-2) .

With these samples deployed we are pretty much done and our pods should now be assigned an ip address. Let’s test it:

```

$: cat <<EOF | kubectl create -f -
---
apiVersion: v1
kind: Pod
metadata:
  name: overlay1pod
spec:
  containers:
  - name: overlay1pod
    image: docker.io/centos/tools:latest
    command:
      - /sbin/init
EOF
```

```
pod/overlay1pod created
```

```
$: kubectl exec overlay1pod ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid_lft forever preferred_lft forever
inet6 ::1/128 scope host
valid_lft forever preferred_lft forever
3: eth0@if17: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
link/ether 1e:92:1f:ac:b9:68 brd ff:ff:ff:ff:ff:ff link-netnsid 0
inet 10.42.2.43/24 scope global eth0
valid_lft forever preferred_lft forever
inet6 fe80::1c92:1fff:feac:b968/64 scope link tentative dadfailed
valid_lft forever preferred_lft forever
```

As you can see we have successfully deployed a pod and were assigned the ip 10.42.2.43 on the eth0 interface, which is the default interface. All extra interfaces will appear as netX, i.e. net1.

### Setting up the Secondary Network

The secondary network needs a few more adjustments and these are all made on the assumption that you want to deploy vxlan. To actually serve a secondary overlay we need to change the VXLAN Identifier “VIN”, which by default is set to 1, and which is now already taken by our first overlay network. So we can change this by configuring the network on our etcd server. We use the clusters own etcd, here marked in green (and we assume that the job runs on the host running the etcd client) and mount in our keys, here marked in red, from the local host which in our case are stored in the /etc/kubernetes/ssl folder.

You can view the entire sample yaml file [HERE](https://neuvector.com/network-security/advanced-kubernetes-networking/#fancyboxID-3) .

Next we can actually deploy the secondary network. The setup of this pretty much looks the same as the primary one, but with some key differences. The most obvious is that we changed the subnet, but we also need to change a few other things.

First of all we need to set a different dataDir, i.e. /var/lib/cni/flannel2, and a different subnetFile, i.e. /run/flannel/flannel2.env. This is needed because they are otherwise occupied and already used by our primary network. Next we need to adjust the bridge because kbr0 is already used by the primary Flannel overlay network.

The remaining configuration required includes changing it to actually target our etcd server which we configured before. In the primary network this was done by connecting to the k8s api directly, which is done via the “–kube-subnet-mgr” flag. But we can’t do that because we also need to modify the prefix from which we want to read. You can see this below marked in orange and settings for our cluster etcd connection in red. The last setting is to specify the subnet file again, marked in green in the sample. Last but not least we add a network definition. The rest of the sample is identically to our main networks config.

See the sample config file for the above steps [HERE](https://neuvector.com/network-security/advanced-kubernetes-networking/#fancyboxID-4) .

Once this is done we have our secondary network ready.

### Assigning Extra Networks

Now that we have a secondary network ready we also need to assign this. To do this we also need to first define a NetworkAttachmentDefinition, which we can use afterwards to assign this network to the container. This is basically the dynamic alternative to the configmap we setup before when initializing multus. This way we can mount the networks we need on demand. In this definition we need to specify the network type, in our case flannel and also necessary configurations. This includes the before mentioned subnetFile, dataDir and bridge name.

The last thing we need to decide for is the name for the network, we name ours flannel.2.

```

$: cat <<EOF | kubectl create -f -
---
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: flannel.2
spec:
  config: '{
    "cniVersion": "0.3.0",
    "type": "flannel",
           "subnetFile": "/run/flannel/flannel2.env",
           "dataDir": "/var/lib/cni/flannel2",
           "delegate": { 
               "bridge": "kbr1" 
           }
  }'
EOF
```

Now we’re finally ready to spawn our first pod with our secondary network.

```

$: cat <<EOF | kubectl create -f -
---
apiVersion: v1
kind: Pod
metadata:
  name: overlay2pod
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
            { "name": "flannel.2" }
    ]'
spec:
  containers:
  - name: overlay2pod
    image: docker.io/centos/tools:latest
    command:
      - /sbin/init
EOF
```

This should create your new pod with your secondary network, and we should see those attached as additional network interfaces now.

```

$: kubectl ~/setup/test/config exec overlay2pod ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid_lft forever preferred_lft forever
inet6 ::1/128 scope host
valid_lft forever preferred_lft forever
3: eth0@if23: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
link/ether 66:c5:7f:01:40:c8 brd ff:ff:ff:ff:ff:ff link-netnsid 0
inet 10.42.0.90/24 scope global eth0
valid_lft forever preferred_lft forever
inet6 fe80::64c5:7fff:fe01:40c8/64 scope link tentative dadfailed
valid_lft forever preferred_lft forever
5: net1@if24: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
link/ether 46:a6:d5:33:54:04 brd ff:ff:ff:ff:ff:ff link-netnsid 0
inet 10.5.22.4/24 scope global net1
valid_lft forever preferred_lft forever
inet6 fe80::44a6:d5ff:fe33:5404/64 scope link
valid_lft forever preferred_lft forever
```

Success, we see the secondary network which was assigned 10.5.22.4 as its ip address.

### Troubleshooting

Should this example not work for you, you will need to look in the logs of your kubelet.  
One common issue is missing CNIs. In my first tests I was missing the bridge CNI since this was not deployed by RKE, fixing this is as easy as  
ing them from the containernetworking repo.

### External Connectivity and Load Balancing

Now that we have a network up and running, the next thing we want is to make our app reachable and configure them to be highly available and scalable. While HA and scalability are not solely achieved by load balancing it is the key component we need to have in place.

Kubernetes has basically four concepts to make an app externally available.

### Using Load Balancers

#### Ingress

An Ingress is basically a loadbalancer with Layer 7 capabilities, specifically HTTP(s). The most commonly used implementation of an ingress controller is the nginx ingress. But this highly depends on your needs and your use case and when in doubt you can default to whichever solution you want to use. For example, traefik or HA Proxy for which ingress controllers already exists. See the traefik guide for an example on how to setup a different ingress controller:  
[https://docs.traefik.io/user-guide/kubernetes/](https://docs.traefik.io/user-guide/kubernetes/)

Configuring an ingress is quite easy. In the following example you see an example of a linked service. In blue you will find the basic configuration which in this example points to a service. In green you find the configuration needed to link your SSL certificate unless you do not employ SSL (you will need this certificate installed before though). And last but not least in brown you will find an example to adjust some of the detailed settings of the nginx ingress. The latter ones you can look up over here: [https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md).

```

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: 12m
  name: my-ingress
  namespace: default
spec:
  rules:
  - host: my.domain.com
    http:
      paths:
      - backend:
          serviceName: api
          servicePort: 5000
        path: /api
      - backend:
          serviceName: websockets
          servicePort: 5000
        path: /ws
  tls:
  - hosts:
    - my.domain.com
```

#### Layer 4 Load Balancer

The layer 4 load balancer, which is defined in kubernetes with type: LoadBalancer, is a service provider dependent load balancing solution. For an on premise machine this will be most probably HA Proxy or a routing solution. Cloud providers may use either their own solution, have special hardware in place or resort to an HA Proxy or a routing solution as well. Should you manage a bare metal installation of a k8s cluster, you might want to give [https://metallb.universe.tf/](https://metallb.universe.tf/) a look.  
The big difference is that the layer 4 load balancer does not understand high level application protocols (layer 7) and is only capable of forwarding traffic. Most of the load balancers on this level also support SSL termination. This typically needs to be configured through annotations and is not standardized. So look this up in the docs of your cloud provider accordingly.

#### Using {host,node}Ports

A {host,node}Port is basically the equivalent to docker -p port:port, especially the hostPort. The nodePort, unlike the hostPort, is available on all nodes instead of only on the nodes running the pod. For nodePort kubernetes creates a clusterIP first and then load balances traffic over this port. The nodePort itself is just an iptable rule to forward traffic on the port to the clusterIP.  
A nodePort is rarely used except in quick testing and only really needed in production if you want every node to expose the port, i.e. for monitoring. Most of the time you will want to use a layer 4 load balancer instead. And hostPort is mostly only really used for testing purposes or very rarely to stick a pod to a specific node and publish under a specific ip address pointing to this node.

To give you an example, a hostPort is defined in the container spec, like the following:

```

        spec:
      containers:
      - args:
        - sleep
        - infinity
        image: ubuntu:xenial
        imagePullPolicy: Always
        name: test
        ports:
        - containerPort: 1111
          hostPort: 1111
          name: 1111tcp11110
          protocol: TCP
```

However a nodePort is defined as a service instead and not in the container spec. An example for a nodePort would look like this:

```

apiVersion: v1
kind: Service
spec:
  externalTrafficPolicy: Cluster
  ports:
  - name: http
    port: 7777
    protocol: TCP
    targetPort: 80
  selector:
    app: MyApp
  sessionAffinity: None
  type: NodePort
```

#### What is a ClusterIp?

A clusterIP is an internally reachable IP for the kubernetes cluster and all services within it. This ip itself load balances traffic to all pods that matches its selector rules. A clusterIP also automatically gets generated in a lot of cases, for example when specifying a type: LoadBalancer service or setting up nodePort. The reason behind this is that all the load balancing happens through the clusterIP.

The clusterIP itself as a concept was created to solve the problem of multiple addressable hosts and the effective renewal of those. It is much easier to have single IP that does not change than to refetch data all the time via service discoveries all the time for all natures of services. Although there are times when it is appropriate to use service discovery instead if you want explicit control, for example in some microservice environments.

### Common Troubleshooting

If you use a public cloud environment and setup the hosts manually your cluster might be missing firewall rules. For example in AWS you will want to adjust your security groups to allow inter-cluster communication as well as ingress and egress. Otherwise this will lead to an inoperable cluster. Make sure you always open the required ports between master and worker nodes. The same goes for ports that you open on the hosts directly, i.e. hostPort or nodePort.

### Network Security

Now that we have setup all of our kubernetes networking, we also need to make sure that we have some security in place. A simple rule in security is to give applications the least access they need. This ensures to a certain degree that even in the case of a security breach attackers will have a hard time to dig deeper into your network. While it does not completely secures you, it makes it harder and more time consuming. This is important because it gives you more time to react and prevent further damage and can often prevent the damage itself. A prominent example is the combination of different exploits/vulnerabilities of different applications, which the attacker is only able to pursue, if there is actually any attack surface reachable from multiple vectors (e.g. network, container, host).

[\[Download this Container Segmentation Guide to Learn Strategies to Protect Workloads\]](https://neuvector.com/network-security/advanced-kubernetes-networking/#fancyboxID-10)

The options here are either we utilize network policies, or we look to 3rd party security solutions for container network security. With network policy we have a solid base to ensure that traffic only flows where it should flow, but this only works for a few CNI’s. They work for example with Calico and Kube-router. Flannel does not support it but luckily as of today you can move to canal, which makes the network policy feature from Calico usable by Flannel. For most other CNI’s there is no support and also no support planned.

But this is not the only issue. The problem is that a network policy rule is only a very simple firewall rule targeting a certain port. This means you can’t apply any advanced settings. For example, you can’t block just a single container on demand should you detect something is suspicious with this container. Further network rules do not understand the traffic, so you don’t have any visibility of the traffic flowing and you’re purely limited to create rules on the Layer 3 and 4 level. And lastly there is no detection of network based threats or attacks such as DDoS, DNS, SQL injection and other damaging network attacks that can occur even over trusted ip addresses and ports.

This is where specialized container network security solutions like [NeuVector](https://neuvector.com/run-time-container-security/) provide the security needed for critical applications such as financial or compliance driven ones. The NeuVector [container firewall](https://neuvector.com/docker-security/how-to-deploy-a-docker-container-firewall/) is a solution that I had experience with deploying at Arvato/Bertelsmann and it provided the layer 7 visibility and protection we needed.

It should be noted that any network security solution must be cloud-native and auto-scaling and adapting. You can’t be checking on iptable rules or having to update anything when you deploy new applications or scale your pods. Perhaps for a simple application stack on a couple nodes you could manage this all manually, but for any enterprise deployment security can’t be slowing down the CI/CD pipeline.

In addition to the security and visibility, I also found that having connection and packet level container network tools helped debug applications during test and staging. With a kubernetes network you’re never really certain where all the packets are going and which pods are being routed to unless you can see the traffic.

### Some Advice on Choosing the Network CNI

Now that kubernetes networking and CNI’s have been covered, one big question always comes up. Which CNI solution should choose? I will try to at least give you advice about how to go about making this decision.

### First, Define the Problem

The first thing for every project is to define the problem you want to solve first in as much detail as possible. You will want to know what kind of applications you want to deploy and what kind of load they generate. Some of the questions you could ask yourself are:

Is my application:

-   Heavy on the network?
-   Sensitive to latency?
-   A monolith?
-   A microservice architected service?
-   On multiple networks?

### Can I Withstand Major Downtime? Or Even Minor?

This is an important question because that you should decide up front, because if you choose one solution now and later you want to switch you will need to re-setup the network and redeploy all your containers. Unless you already have something in place like multus and work with multiple networks, this will mean a downtime for your service. Most of the time it will be fine if you have a planned maintenance window, but as you grow, zero downtime becomes more important.

### My Application is on Multiple Networks

This scenario is quite common in on-premise installations. In fact, if you only want to separate the traffic going over the private network and the public network this will require you to either setup multiple networks or have clever routing.

#### Is There a Certain Feature You Need from the CNI’s?

Another thing influencing your decision can be that you want some features of the CNI’s not available to other CNI’s. For example you want to utilize weave or you want more mature loadbalancing through ipvs.

### What Network Performance is Required?

If you answered that your application is sensitive to latency or heavy on the network you might want to avoid any overlay network. Overlays can can be expensive on performance and even more so at scale. If this is the case the only way to improve performance on the network is to avoid overlays and utilize networking utilities like routing instead. When you look for network performance you have a few choices, for example:

#### Ipvlan

Ipvlan could be an option for you, it has a good performance, but has it caveats, i.e. you can’t use macv{tap,lan} at the same time on the same host.

#### Calico

#### Calico is not the most user friendly CNI, but it provides you with much better performance compared to vxlan and can be scaled without issues.

Kube-Router

Kube-router will give you better performance like Calico, since they both use BGP and routing, plus support for LVS/IPVS. But it might not be as battle tested as Calico.

#### Cloud Provider Solutions

Last but not least, some of the cloud providers do provide own kubernetes networking solutions that may or may not perform better. Amazon for example has their aws-vpc which is supported on flannel. Aws-vpc performs in most scenarios about as good as ipvlan.

### But I Just Want Something That Works!

Yes, that is understandable, and this is the case for most users. In this case probably canal or Flannel with vxlan will be the way to go because it is a no-brainer and just works. However as I said before vxlan is slow and it will cost you significantly more resources as you grow. But this is definitely the easiest way to start.

### Just Make a Decision

It really is a matter of making a decision rather than making none at all. If you don’t have specific features requirements it is fine to start with Flannel and vxlan. It will involve some work to migrate later if you’ve deployed to production but it is better to make decision that is wrong in the long term than making no decision at all.

![[Raw/Media/Resources/f752a83eb192a91cfaacd506ca4f29b9_MD5.webp]]With all this information, I hope that you will have some relevant background and a good understanding of how kubernetes networking works.

To learn more about the attack surface of Kubernetes workloads and how to protect the CI/CD pipeline, Kubernetes system services, and containers at run-time, [Download this Ultimate Guide to Kubernetes Security](https://neuvector.com/network-security/advanced-kubernetes-networking/#fancyboxID-7)