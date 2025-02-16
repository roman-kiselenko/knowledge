---
title: Kubernetes LoadBalance service using Cilium BGP control plane
source: https://medium.com/@valentin.hristev/kubernetes-loadbalance-service-using-cilium-bgp-control-plane-8a5ad416546a
clipped: 2025-02-12
published: 
category: network
tags:
  - network
read: false
---

The Kubernetes container orchestration platform offers plugin support for Load Balancers, making it possible to create highly available services with even traffic distribution across a set of containers. One of the challenges facing those looking to experiment with Kubernetes is the lack of a “builtin” Load Balancer.

At a minimum, a Kubernetes cluster requires a container runtime interface \[ **CRI**\] plugin (like Containerd or CRI-O) and a container networking interface \[ **CNI**\] plugin (like Calico, Flannel, Weave or Cilium). While installing CRI and CNI tools is generally straightforward, adding support for external load balancer services to a cluster has traditionally been a complex task.

The good news is that Cilium, a popular CNI plugin, has recently added LoadBalancer IP Address Management (LB IPAM) for Kubernetes. This combined with Ciliums’s blazing fast XDP packet processing makes Cilium a great choice for building out a fully featured Kubernetes cluster.

In this blog, we will configure Cilium to supply Load Balancer service support in Kubernetes. Our solution will allow our users to create external load balancer services for pods running in our cluster (North -> South). We’ll configure our solution on a set of Raspberry Pis running K3s, a certified Kubernetes distribution designed for unattended / resource-constrained systems.

Picture source: [https://cilium.io](https://cilium.io/)

![[Raw/Media/Resources/4548cff827a88aa907043f5dbc173fd9_MD5.png]]

We will use four RaspberryPi 4s for our cluster. One of the Pis will act as the **Control Plane Node**, and the remaining three nodes will serve as **Worker Nodes**. As you can see, this particular RaspberryPi rack has a colorful disposition.

Rack: [https://www.uctronics.com/cluster-and-rack-mount/uctronics-upgraded-complete-enclosure-raspberry-pi-cluster.html](https://www.uctronics.com/cluster-and-rack-mount/uctronics-upgraded-complete-enclosure-raspberry-pi-cluster.html)

Our network topology is straightforward with a single host subnet, 192.168.1.0/24, for all of the nodes.

![[Raw/Media/Resources/06c98b502389fe7e0e46cb2ec826ee64_MD5.png]]

![[Raw/Media/Resources/aa32bd6d11a497754d10e83ec8a2d019_MD5.png]]

We will start with a **minimal k3s installation**. K3s is designed to be a complete solution and installs several networking components by default, including:

-   Flannel CNI plugin
-   Traefik Ingress Controller
-   Metallb Load Balancer
-   Kubernetes Metrics Servers
-   Kube-proxy in cluster service proxy

For our setup, we will disable all of these addons. Many of them will be replaced with more integrated (and often faster/more efficient **Cilium** implementations), and others we will install ourselves.

Let’s check the nodes (n.b. we use ‘k’ as an alias for ‘kubectl’)

➜ k3s-rpi (main) ✗ k get nodes  
NAME                   STATUS      ROLES                  AGE     VERSION  
k3s-worker-node-03     NotReady    <none>                 3d18h   v1.27.3+k3s1  
k3s-worker-node-02     NotReady    <none>                 3d18h   v1.27.3+k3s1  
k3s-worker-node-01     NotReady    <none>                 3d18h   v1.27.3+k3s1  
k3s-control-plane-01   NotReady    control-plane,master   3d18h   v1.27.3+k3s1

You can see that all nodes are in status **NotReady**. This is because we have yet to install a CNI plugin and normal pods can not run without a pod network. Let’s check the pods. To start clean, we delete every single pod, even those from the **kube-system.**

✗ k get pods -A  
NAMESPACE            NAME                                                    READY   STATUS    RESTARTS   AGE  
kube-system          coredns-5d78c9869d-p5svm                                0/1     Pending   0          43s  
local-path-storage   local-path-provisioner-6bc4bddd6b-xpn8h                 0/1     Pending   0          84s

In a K3s cluster we don’t see the kube-apiserver, scheduler, datastore or controller-manager Pods because everything is packaged together in one binary where they all run together.

From the output, we can see some pods are **Pending**. Without a CNI plugin, we don’t have a Pod network. Let’s install Cilium to get our network running!

While there are several ways you can install Cilium, we will use the **cilium cli**. **Helm** is another popular option. Here is the official documentation for [Cilium cli installation](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli)

The example here uses a Mac as the administration and setup machine. You can, of course, use Linux and other systems as well, for more information on Cilium setup from other systems, refer to the official documentation.

CILIUM\_CLI\_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)  
CLI\_ARCH=amd64  
if \[ "$(uname -m)" = "arm64" \]; then CLI\_ARCH=arm64; fi  
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM\_CLI\_VERSION}/cilium-darwin-${CLI\_ARCH}.tar.gz{,.sha256sum}  
shasum -a 256 -c cilium-darwin-${CLI\_ARCH}.tar.gz.sha256sum  
sudo tar xzvfC cilium-darwin-${CLI\_ARCH}.tar.gz /usr/local/bin  
rm cilium-darwin-${CLI\_ARCH}.tar.gz{,.sha256sum}

We can use the Cilium version command to verify the CLI is properly installed:

✗ cilium version  
cilium-cli: v0.15.4 compiled with go1.20.4 on darwin/arm64  
cilium image (default): v1.13.4  
cilium image (stable): v1.13.4  
cilium image (running): unknown. Unable to obtain cilium version, no cilium pods found in namespace "kube-system"

Let’s now install the **cilium CNI**:

✗ cilium install  
🔮 Auto-detected Kubernetes kind: k3s  
ℹ️  Using Cilium version 1.13.4  
🔮 Auto-detected cluster name: k3s-rpi  
ℹ️  kube-proxy-replacement disabled  
🔮 Auto-detected datapath mode: tunnel  
🔮 Auto-detected kube-proxy has not been installed  
ℹ️  Cilium will fully replace all functionalities of kube-proxy

Let’s recheck the pods. As you can see, there are new players in the game with the **cilium-prefix**.

✗ k get pods \-A  
NAMESPACE            NAME                                                    READY   STATUS     RESTARTS   AGE  
kube\-system          cilium\-6fnjv                                            0/1     Init:1/6   0          20s  
kube\-system          cilium\-g2s9w                                            0/1     Init:4/6   0          20s  
kube\-system          cilium\-hgftv                                            0/1     Init:1/6   0          20s  
kube\-system          cilium\-operator\-768959858c\-zjjnc                        1/1     Running    0          20s  
kube\-system          cilium\-p82qr                                            0/1     Init:3/6   0          20s  
kube\-system          coredns\-5d78c9869d\-p5svm                                0/1     Pending    0          5m13s  
local\-path\-storage   local\-path\-provisioner\-6bc4bddd6b\-q8f2x                 0/1     Pending    0          5m13s

Here are the two components we see in our cluster. We put a small explanation about them.

> [*cilium-operator*](https://docs.cilium.io/en/stable/internals/cilium_operator/)*: The Cilium Operator is responsible for managing duties in the cluster which should logically be handled once for the entire cluster rather than once for each node in the cluster*
> 
> [*Cilium*](https://docs.cilium.io/en/v1.5/concepts/#cilium-agent) *pods: The Cilium agent (cilium-agent) runs on each Linux container host. At a high level, the agent accepts configuration that describes service-level network security and visibility policies. It then listens to events in the container runtime to learn when containers are started or stopped, and it creates custom BPF programs, which the Linux kernel uses to control all network access in/out of those containers.*

You can see that there is no **agent** in the name of the pod, but if you check the pod logs, you can see that the pod **cilium-xxxxx** is a [cilium-agent](https://docs.cilium.io/en/stable/cmdref/cilium-agent/#cilium-agent).

✗ k logs -n kube-system cilium\-6fnjv | head \-1  
Defaulted container "cilium-agent" out of: cilium-agent, config (init), mount-cgroup (init), apply-sysctl-overwrites (init), mount-bpf-fs (init), clean-cilium-state (init), install-cni-binaries (init)

Give it some time to settle down and check the pods again.

✗ k get pods -A  
NAMESPACE            NAME                                                    READY   STATUS    RESTARTS   AGE  
kube-system          cilium-6fnjv                                            1/1     Running   0          6m3s  
kube-system          cilium-g2s9w                                            1/1     Running   0          6m3s  
kube-system          cilium-hgftv                                            1/1     Running   0          6m3s  
kube-system          cilium-operator-768959858c-zjjnc                        1/1     Running   0          6m3s  
kube-system          cilium-p82qr                                            1/1     Running   0          6m3s  
kube-system          coredns-5d78c9869d-p5svm                                1/1     Running   0          10m  
local-path-storage   local-path-provisioner-6bc4bddd6b-q8f2x                 1/1     Running   0          10m

Everything is in a **Running** state, our cluster is working like a charm.

Picture soruce: [https://cilium.io](https://cilium.io/)

![[Raw/Media/Resources/b583dfd8eb01301c5c227803bd0947fb_MD5.png]]

Let’s switch gears and move on to BGP (Border Gateway Protocol).

**What is a BGP ?**: BGP is an internet routing protocol that enables the exchange of routing information between autonomous systems (ASes), allowing networks to learn and advertise routes to reach different destinations over public and private networks.

For more information on BGP, take a look at [RFC 4271 — BGP](https://datatracker.ietf.org/doc/html/rfc4271).

From the official [Cilium BGP control plane](https://docs.cilium.io/en/stable/network/bgp-control-plane/) documentation, you will see that currently, a single flag in the Cilium agent exists to turn on the **BGP Control Plane** feature set. There are different ways to enable this flag, however we will continue using the cilium cli (Helm requires a different approach, so check the official documentation if you are using Helm).

Before we change the BGP flag, let’s check the current configuration.

\> n.b. You can enable BGP when you install Cilium, but we want to show you each of the underlying steps.

✗ cilium config view | grep -i bgp  
enable-bgp-control-plane                       false

As you can see, the BGP Control Plane feature is disabled by default. Let’s enable it!

✗ cilium config set enable-bgp-control-plane true  
✨ Patching ConfigMap cilium-config with enable-bgp-control-plane=true...  
♻️  Restarted Cilium pods

Let’s check the config to verify:

✗ cilium config view | grep -i bgp  
enable-bgp-control-plane   

Now it looks better. Let’s check on our pods.

✗ k get pods -n kube-system  
NAME                                                    READY   STATUS    RESTARTS   AGE  
cilium-5mczq                                            0/1     Running   0          9s  
cilium-k9p6z                                            0/1     Running   0          9s  
cilium-operator-768959858c-zjjnc                        1/1     Running   0          21m  
cilium-zg5dw                                            0/1     Running   0          9s  
cilium-zlg96                                            0/1     Running   0          9s  
coredns-5d78c9869d-p5svm                                1/1     Running   0          26m  
local-path-provisioner-6bc4bddd6b-q8f2x                 1/1     Running   0          10m

The READY state for our Cilium Agents is **0/1** , which means there’s a problem. Let’s read the logs to see why the Cilium Agents are no longer READY.

✗ k logs -n kube-system cilium-5mczq | tail -1  
Defaulted container "cilium-agent" out of: cilium-agent, config (init), mount-cgroup (init), apply-sysctl-overwrites (init), mount-bpf-fs (init), clean-cilium-state (init), install-cni-binaries (init)  
level=error msg=k8sError error="github.com/cilium/cilium/pkg/k8s/resource/resource.go:183: Failed to watch \*v2alpha1.CiliumBGPPeeringPolicy: failed to list \*v2alpha1.CiliumBGPPeeringPolicy: the server could not find the requested resource (get ciliumbgppeeringpolicies.cilium.io)" subsys=k8s

Hmmm `**Failed to watch *v2alpha1.CiliumBGPPeeringPolicy**`. Setting **enable-bgp-control-plane true** causes the Cilium Agents to look for the Cilium BGP Peering Policy, which does not yet exist.

Kubernetes is highly extensible, allowing tools to create their own resource types. Cilium uses Kubernetes CRDs (Custom Resource Definitions) to define most of its configuration objects. The Cilium Operator did not create the CiliumBGPPeeringPolicy CRD because we were not using that feature at installation time. Let's check the resource types defined in our cluster with the **api-resources** command:

✗ k api-resources | grep -i cilium  
ciliumclusterwidenetworkpolicies   ccnp                                cilium.io/v2                           false        CiliumClusterwideNetworkPolicy  
ciliumendpoints                    cep,ciliumep                        cilium.io/v2                           true         CiliumEndpoint  
ciliumexternalworkloads            cew                                 cilium.io/v2                           false        CiliumExternalWorkload  
ciliumidentities                   ciliumid                            cilium.io/v2                           false        CiliumIdentity  
ciliumloadbalancerippools          ippools,ippool,lbippool,lbippools   cilium.io/v2alpha1                     false        CiliumLoadBalancerIPPool  
ciliumnetworkpolicies              cnp,ciliumnp                        cilium.io/v2                           true         CiliumNetworkPolicy  
ciliumnodeconfigs                                                      cilium.io/v2alpha1                     true         CiliumNodeConfig  
ciliumnodes                        cn,ciliumn                          cilium.io/v2                           false        CiliumNode

No **CiliumBGPPeeringPolicy**!  
Looking closely at the pods, we can see that the Cilium Agents were redeployed, but the Cilium Operator was not. In the example above, the Operator has been running for **21** **minutes**, but the Agents have been running for only a few seconds. Our cilium-operator was not redeployed when we updated the configuration and thus has not taken any action to support BGP. So we need to refresh the Operator.

✗ k get pods \-n kube\-system  
NAME                                                    READY   STATUS    RESTARTS   AGE  
cilium\-5mczq                                            0/1     Running   0          9s  
cilium\-k9p6z                                            0/1     Running   0          9s  
cilium\-operator\-768959858c\-zjjnc                        1/1     Running   0          21m

Let’s redeploy our **cilium-operator** by deleting it, this will cause the new instance to read the updated configuration on startup.

✗ k delete pod -n kube-system cilium-operator\-768959858c-zjjnc

Now let’s take a look at the Operator logs:

✗ k logs -n kube-system cilium-operator-768959858c-zk7zq | grep CRD  
level=info msg="Creating CRD (CustomResourceDefinition)..." name=CiliumBGPPeeringPolicy/v2alpha1 subsys=k8s  
level=info msg="CRD (CustomResourceDefinition) is installed and up-to-date" name=CiliumExternalWorkload/v2 subsys=k8s  
level=info msg="CRD (CustomResourceDefinition) is installed and up-to-date" name=CiliumNodeConfig/v2alpha1 subsys=k8s  
level=info msg="CRD (CustomResourceDefinition) is installed and up-to-date" name=CiliumLoadBalancerIPPool/v2alpha1 subsys=k8s  
level=info msg="CRD (CustomResourceDefinition) is installed and up-to-date" name=CiliumEndpoint/v2 subsys=k8s  
level=info msg="CRD (CustomResourceDefinition) is installed and up-to-date" name=CiliumIdentity/v2 subsys=k8s  
level=info msg="CRD (CustomResourceDefinition) is installed and up-to-date" name=CiliumNode/v2 subsys=k8s  
level=info msg="CRD (CustomResourceDefinition) is installed and up-to-date" name=CiliumClusterwideNetworkPolicy/v2 subsys=k8s  
level=info msg="CRD (CustomResourceDefinition) is installed and up-to-date" name=CiliumNetworkPolicy/v2 subsys=k8s  
level=info msg="CRD (CustomResourceDefinition) is installed and up-to-date" name=CiliumBGPPeeringPolicy/v2alpha1 subsys=k8s  
level=info msg="Starting CRD identity garbage collector" interval=15m0s subsys=cilium-operator-generic

As you can see, the operator has created the needed CRD: `**Creating CRD (CustomResourceDefinition)..." name=CiliumBGPPeeringPolicy/v2alpha1**`

Redisplay the api-resources to see the new CRD:

✗ k api-resources | grep -i ciliumBGP  
ciliumbgppeeringpolicies    bgpp    cilium.io/v2alpha1                     false        CiliumBGPPeeringPolicy

Now that we have a **CiliumBGPPeeringPolicy** type (CRD) we can create an object of that type to define our Cilium BGP peering policy.

Here is the yaml file which we will use to create it.

✗ cat cilium-bgp-policy.yaml

apiVersion: "cilium.io/v2alpha1"  
kind: CiliumBGPPeeringPolicy  
metadata:  
 name: 01\-bgp-peering-policy  
spec:  
 nodeSelector:  
   matchLabels:  
     bgp-policy: a  
 virtualRouters:  
 \- localASN: 64512  
   exportPodCIDR: true  
   neighbors:  
    \- peerAddress: '192.168.1.1/32'  
      peerASN: 64512  
   serviceSelector:  
     matchExpressions:  
       \- {key: somekey, operator: NotIn, values: \['never-used-value'\]}"cilium.io/v2alpha1" kind: CiliumBGPPeeringPolicy metadata: name: 01-bgp-peering-policy spec: nodeSelector: matchLabels: bgp-policy: a virtualRouters: - localASN: 64512 exportPodCIDR: true neighbors: - peerAddress: '192.168.1.1/32' peerASN: 64512 serviceSelector: matchExpressions: \- {key: somekey, operator: NotIn, values: \['never-used-value'\]}

Let’s break it down and discuss the options section by section. The first part of our specification (\`spec:\`) is the nodeSelector. This defines which nodes our policy applies to. The label **bgp-policy=a** is defined here, so we will have to add this label to all of our cluster nodes before the policy will be applied to them.

 nodeSelector:  
   matchLabels:  
     bgp-policy: a

Next, we define our virtual routers. Virtual routers allow multiple distinct routers to be supported within a single routed environment, allowing clusters to configure multiple, separate logical routers within a single network of nodes.

The Cilium cluster uses a local AS number (ASN) to identify the AS in which the BGP service resides. For our purposes, we’ll use an ASN in the well-known private ASN range (64512–65535). We set our ASN to **64512**:

 virtualRouters:  
 \- localASN: 64512

Next, we ask Cilium to advertise the Pod Network to peers to allow external traffic to be routed directly to our pods (you can disable this feature if not desired). Documentation can be found [here](https://docs.cilium.io/en/latest/network/bgp-control-plane/):

exportPodCIDR: true

Next, we set the BGP peer we will communicate with. This is typically the upstream router. In our case, this is our **Mikrotik router**, on which we will configure BGP later. In the example, we specify the same ASN as our nodes and the IP of our router: **192.168.1.1/32**.

   neighbors:  
    \- peerAddress: '192.168.1.1/32'  
      peerASN: 64512

The final part of our policy uses the `serviceSelector` key to define which services we will expose. The serviceSelector allows you to configure which Kubernetes Load Balancer Services are advertised (announced) outside of the Cluster (documentation can be found [here](https://docs.cilium.io/en/stable/network/bgp-control-plane/#service-announcements): ). This is the relevant section:

*Service announcements*

> By default, virtual routers will not announce services. Virtual routers will advertise *the ingress IPs of any LoadBalancer services that match* the .serviceSelector of the virtual router. If you wish to announce ALL services within the cluster, a NotIn match *expression with a dummy key and value can be used like:\**
> 
> node selector: Nodes which are selected by this label selector will apply the given policy

We want all of our Load Balancer Services to be available externally so we will create a dummy selector to select **all services**. You can also use the selector setting to select specific services by label or limit the functionality to specific namespaces.

serviceSelector:  
     matchExpressions:  
       \- {key: somekey, operator: NotIn, values: \['never-used-value'\]}

Now that we have an understanding of the policy, let’s apply it to the cluster:

✗ k apply -f cilium-bgp-policy.yaml  
ciliumbgppeeringpolicy.cilium.io/01-bgp-peering-policy created

We need to **label** the nodes that we want the BGP policy to apply to. In our case, we will label all worker nodes, leaving out the control-plane node. Our **CiliumBGPPeeringPolicy** node selector expects the **bgp-policy=a** label.

✗ k label nodes k3s-worker-node-01 bgp-policy=a  
✗ k label nodes k3s-worker-node-02 bgp-policy=a  
✗ k label nodes k3s-worker-node-03 bgp-policy=a

Select all nodes with the **bgp-policy=a** label to make sure the label is applied properly:

✗ k3s-rpi (main) ✗ k get nodes -l bgp-policy=a  
NAME                 STATUS   ROLES    AGE     VERSION  
k3s-worker-node-03   Ready    <none>   3d19h   v1.27.3+k3s1  
k3s-worker-node-02   Ready    <none>   3d19h   v1.27.3+k3s1  
k3s-worker-node-01   Ready    <none>   3d19h   v1.27.3+k3s1

When you create a Load Balancer Service in a Kubernetes cluster, the cluster itself does not actually assign the Service a Load Balancer IP (aka External IP), we need a plugin to do that. If you create a Load Balancer Service without a Load Balancer plugin the External IP address will show **Pending** indefinitely.

The Cilium [LoadBalancer IP Address Management (LB IPAM)](https://docs.cilium.io/en/stable/network/lb-ipam/#loadbalancer-ip-address-management-lb-ipam) feature can be used to provision IP addresses for our Load Balancer Services.

Here is what the official doc says about it:

> LB IPAM is a feature that allows Cilium to assign IP addresses to Services of type LoadBalancer. This functionality is usually left up to a cloud provider, however, when deploying in a private cloud environment, these facilities are not always available.
> 
> *This section must understand that* LB IPAM is always enabled but dormant. The controller is awoken when the first IP Pool is added to the cluster.

Let’s create our **cilium LoadBalancer IP pool**. To create a pool we name it and give a CIDR range. We’ll use **172.198.1.0/24** as our CIDR range, it is important that this range does not overlap with other networks in use with your cluster.

✗ cat cilium-ippool.yaml

apiVersion: "cilium.io/v2alpha1"  
kind: CiliumLoadBalancerIPPool  
metadata:  
  name: "lb-pool"  
spec:  
  cidrs:  
  - cidr: "172.198.1.0/24"apirsion: "cilium.io/v2alpha1" kind: CiliumLoadBalancerIPPool metadata: name: "lb-pool" spec: cidrs: - cidr: "172.198.1.0/24"

Let’s create it.

✗ k create -f cilium-ippool.yaml  
ciliumloadbalancerippool.cilium.io/lb-pool created

The [LoadBalancer IP Address Management (LB IPAM)](https://docs.cilium.io/en/stable/network/lb-ipam/#loadbalancer-ip-address-management-lb-ipam) documentation provides several additional examples.

> *IP Pools are not allowed to have overlapping CIDRs. When an administrator creates pools that overlap, a soft error is caused. The last added pool will be marked as Conflicting, and no further allocation will happen from that pool.*

That is all you need to do to enable Load Balancer IPAM in Kubernetes.

The **cilium cli** provides a number of useful commands for checking BGP status. Use the “ **peers** “ command to display BGP Peer information:

✗ cilium bgp peers  
Node                 Local AS   Peer AS   Peer Address   Session State   Uptime     Family         Received   Advertised  
k3s-worker-node-01   64512      64512     192.168.1.1    active                     ipv4/unicast   0          0  
                                                                                    ipv6/unicast   0          0  
k3s-worker-node-02   64512      64512     192.168.1.1    active                     ipv4/unicast   0          0  
                                                                                    ipv6/unicast   0          0  
k3s-worker-node-03   64512      64512     192.168.1.1    active                     ipv4/unicast   0          0

From the output, we can see that our peers are configured on the k3s side and that our Session State is “active”. This means that our nodes are configured but that they have not been able to establish a connection with a peer. This is expected because we have not configured the upstream router to peer with the nodes yet.

Let’s configure our router to complete the installation.

Now we need to configure our north/south router to create a BGP session between it and the Kubernetes worker nodes. This solution uses a **Mikrotik router**, here is the [official Mikrotik BGP documentation.](https://help.mikrotik.com/docs/display/ROS/BGP)

Let’s ssh to the router and configure it. You can do that from the GUI but we will use the CLI.

✗ ssh 192.168.1.1  
  MMM      MMM       KKK                          TTTTTTTTTTT      KKK  
  MMMM    MMMM       KKK                          TTTTTTTTTTT      KKK  
  MMM MMMM MMM  III  KKK  KKK  RRRRRR     OOOOOO      TTT     III  KKK  KKK  
  MMM  MM  MMM  III  KKKKK     RRR  RRR  OOO  OOO     TTT     III  KKKKK  
  MMM      MMM  III  KKK KKK   RRRRRR    OOO  OOO     TTT     III  KKK KKK  
  MMM      MMM  III  KKK  KKK  RRR  RRR   OOOOOO      TTT     III  KKK  KKK

  MikroTik RouterOS 7.8 (c) 1999-2023       https://www.mikrotik.com/

Press F1 for help  
\[vhristev@MikroTik\]  
\[vhristev@MikroTik\] > /routing/bgp/connection/  
\[vhristev@MikroTik\] /routing/bgp/connection> add address-families=ip as=64512 disabled=no local.role=ibgp name=PEER\_TO\_K3S\_WN\_1 output.default-originate=always remote.address=192.168.1.201 routing-table=main  
\[vhristev@MikroTik\] /routing/bgp/connection> add address-families=ip as=64512 disabled=no local.role=ibgp name=PEER\_TO\_K3S\_WN\_2 output.default-originate=always remote.address=192.168.1.202 routing-table=main  
\[vhristev@MikroTik\] /routing/bgp/connection> add address-families=ip as=64512 disabled=no local.role=ibgp name=PEER\_TO\_K3S\_WN\_3 output.default-originate=always remote.address=192.168.1.203 routing-table=main

In the session above we connect to the router, navigate to **/routing/bgp/connection/** a special Mikrotik CLI configuration directory, and from there, we create three BGP connections. Here are the details:

-   **address-families=ip**: We want just ipv4
-   **as=64512**: Our BGP ASN
-   **disabled=no**: Connection is active (NOT disabled)
-   **local.role=ibgp**: We use ibgp because it’s a local (internal). For external ASNs we can use ebgp
-   **name=PEER\_TO\_K3S\_WN\_1**: Name of the connection, in our case k3s worker node 01, 02 and 03 respectively
-   **output.default-originate=always**: We want to advertise the default route
-   **remote.address=192.168.1.201**: IP address of the peer a.k.a. our k3s worker node
-   **routing-table=main**: Use the main routing table

We are all done with the router. Now let’s check the BGP peers on the Cilium side.

✗ cilium bgp peers  
Node                 Local AS   Peer AS   Peer Address   Session State   Uptime     Family         Received   Advertised  
k3s-worker-node-01   64512      64512     192.168.1.1    established     1h33m42s   ipv4/unicast   1          2  
                                                                                    ipv6/unicast   0          0  
k3s-worker-node-02   64512      64512     192.168.1.1    established     1h33m37s   ipv4/unicast   1          2  
                                                                                    ipv6/unicast   0          0  
k3s-worker-node-03   64512      64512     192.168.1.1    established     1h33m47s   ipv4/unicast   1          2

Now we can see Cilium has sessions **established**, and we have **Received** and **Advertised** routes. Let’s check the router:

\[vhristev@MikroTik\] /routing/bgp/connection\> ..  
\[vhristev@MikroTik\] /routing/bgp\> session/print  
Flags: E \- established  
 0 E name\="PEER\_TO\_K3S\_WN\_2-1"  
     remote.address\=192.168.1.202 .as\=64512 .id\=192.168.1.202 .capabilities\=mp,rr,enhe,as4,fqdn .afi\=ip,ipv6 .hold\-time\=1m30s .messages\=1126 .bytes\=21428 .eor\=""  
     local.role\=ibgp .address\=192.168.1.1 .as\=64512 .id\=192.168.1.1 .capabilities\=mp,rr,gr,as4 .messages\=1127 .bytes\=21440 .eor\=""  
     output.procid\=21 .default\-originate\=always  
     input.procid\=21 .last\-notification\=ffffffffffffffffffffffffffffffff0015030603 ibgp  
     multihop\=yes hold\-time\=1m30s keepalive\-time\=30s uptime\=1h33m44s560ms last\-started\=jul/26/2023 15:00:56 last\-stopped\=jul/26/2023 14:50:58

 1 E name\="PEER\_TO\_K3S\_WN\_1-1"  
     remote.address\=192.168.1.201 .as\=64512 .id\=192.168.1.201 .capabilities\=mp,rr,enhe,as4,fqdn .afi\=ip,ipv6 .hold\-time\=1m30s .messages\=1128 .bytes\=21466 .eor\=""  
     local.role\=ibgp .address\=192.168.1.1 .as\=64512 .id\=192.168.1.1 .capabilities\=mp,rr,gr,as4 .messages\=1129 .bytes\=21478 .eor\=""  
     output.procid\=20 .default\-originate\=always  
     input.procid\=20 ibgp  
     multihop\=yes hold\-time\=1m30s keepalive\-time\=30s uptime\=1h33m49s620ms last\-started\=jul/26/2023 14:59:51

 2 E name\="PEER\_TO\_K3S\_WN\_3-1"  
     remote.address\=192.168.1.203 .as\=64512 .id\=192.168.1.203 .capabilities\=mp,rr,enhe,as4,fqdn .afi\=ip,ipv6 .hold\-time\=1m30s .messages\=192 .bytes\=3682 .eor\=""  
     local.role\=ibgp .address\=192.168.1.1 .as\=64512 .id\=192.168.1.1 .capabilities\=mp,rr,gr,as4 .messages\=193 .bytes\=3694 .eor\=""  
     output.procid\=22 .default\-originate\=always  
     input.procid\=22 ibgp  
     multihop\=yes hold\-time\=1m30s keepalive\-time\=30s uptime\=1h33m54s650ms last\-started\=jul/26/2023 14:59:21

We can also inspect the status from the Mikrotik GUI. The Connections tab shows the configured connections:

![[Raw/Media/Resources/1880d24137044c5719c68ba0f550d42f_MD5.png]]

The Sessions tab shows the active Mikrotik BGP sessions with the Kubernetes nodes, complete with uptime.

![[Raw/Media/Resources/3cd6306328a87307e8299eef0be2b45e_MD5.png]]

Beautiful, we have three established sessions. Now let’s check the routes that are being **advertised**.

\[vhristev@MikroTik\] /routing/bgp/advertisements> print  
 0 peer=PEER\_TO\_K3S\_WN\_3-1 dst=0.0.0.0/0 local-pref=100 nexthop=78.130.232.1 origin=0

 0 peer=PEER\_TO\_K3S\_WN\_2-1 dst=0.0.0.0/0 local-pref=100 nexthop=78.130.232.1 origin=0

 0 peer=PEER\_TO\_K3S\_WN\_1-1 dst=0.0.0.0/0 local-pref=100 nexthop=78.130.232.1 origin=0

Finally, take a look at the Mikrotik routing table:

\[vhristev@MikroTik\] /routing/bgp> /routing/route/print  
Flags: X, F, A - ACTIVE; c, s, b, d, a - SLAAC; H - HW-OFFLOADED  
Columns: DST-ADDRESS, GATEWAY, AFI, DISTANCE, SCOPE, TARGET-SCOPE, IMMEDIATE-GW  
    DST-ADDRESS       GATEWAY        AFI   DISTANCE  SCOPE  TARGET-SCOPE  IMMEDIATE-GW  
Xs  192.168.1.202/32  bridge                      1     30            10  
Xs  192.168.1.201/32  bridge                      1     30            10  
Xs  192.168.1.203/32  bridge                      1     30            10  
Ad  0.0.0.0/0         78.130.232.1   ip4          1     30            10  78.130.232.1%ether5  
Ab  10.136.1.0/24     192.168.1.201  ip4        200     40            30  192.168.1.201%bridge  
Ab  10.136.2.0/24     192.168.1.203  ip4        200     40            30  192.168.1.203%bridge  
Ab  10.136.3.0/24     192.168.1.202  ip4        200     40            30  192.168.1.202%bridge  
Ac  78.130.232.0/24   ether5         ip4          0     10                ether5  
 b  172.198.1.193/32  192.168.1.203  ip4        200     40            30  192.168.1.203%bridge  
 b  172.198.1.193/32  192.168.1.202  ip4        200     40            30  192.168.1.202%bridge  
Ab  172.198.1.193/32  192.168.1.201  ip4        200     40            30  192.168.1.201%bridge

The last three lines, show an IP **172.198.1.193/32** from our **LoadBalancer** External ip pool **172.198.1.0/24**.

So far, so good. Now let’s create a **pod** with a **service** type **LoadBalancer** and test it.

We are going to make a simple nginx **pod** and a simple **service** exposing port 8080, with type **LoadBalancer**. This should cause Cilium to provision an external IP for our logical load balancer and then advertise the route through BGP.

✗ cat pod.yaml service.yaml

  
apiVersion: v1  
kind: Pod  
metadata:  
  name: simple-pod  
  labels:  
    app: simple-pod  
spec:  
  containers:  
  \- name: my-app-container  
    image: nginx:latest  
    ports:  
    \- containerPort: 80

  
apiVersion: v1  
kind: Service  
metadata:  
  name: my-service  
spec:  
  selector:  
    app: simple-pod    
  ports:  
  \- protocol: TCP  
    port: 8080  
    targetPort: 80  
  type: LoadBalancer

Let’s create it:

✗ k apply -f pod.yaml  
pod/simple-pod created  
✗ k apply -f service.yaml  
service/my-service created

Let’s see what we have. From the output below, we know that we have a **running pod** with the name simple-pod and service with the name my-service, but the most crucial part is that we have a service TYPE **LoadBalancer** with **EXTERNAL-IP** from our **ip-pool**, which we created earlier, and we get **172.198.1.246** (in the example, your IP may vary).

➜ k3s-rpi (main) ✗ k get pod,svc  
NAME             READY   STATUS    RESTARTS   AGE  
pod/simple-pod   1/1     Running   0          41s

NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE  
service/kubernetes   ClusterIP      172.20.0.1      <none>          443/TCP          3d21h  
service/my-service   LoadBalancer   172.20.32.184   172.198.1.246   8080:30232/TCP   27s

You can see the Load Balancer IP Pool CIDR by displaying the **CiliumLoadBalancerIPPool** object:

✗ kubectl get CiliumLoadBalancerIPPool lb-pool -o jsonpath='{.spec.cidrs\[0\].cidr}'  
172.198.1.0/24%

Now, from an external machine, my laptop, which is North of the router, we can reach.

✗ curl  172.198.1.246:8080  
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

In this post we walked through the process of creating Cilium based support for Load Balancer Services in a minimal K3s Kubernetes cluster. Hopefully this step by step approach has given you a better understanding of the network level operations involved in Kubernetes Load Balancers and a spring board with which to start your own experiments. Thanks for reading!