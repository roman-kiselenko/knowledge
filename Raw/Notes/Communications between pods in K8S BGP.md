---
title: Communications between pods in K8S
source: https://medium.com/@Kubway/communications-between-pods-in-k8s-2a952d409581
clipped: 2025-03-01
published: 
category: network
tags:
  - k8s
read: false
---


**Author : Marie F. from Kubway.**

In this 2 parts story, we see how communications between pods in Kubernetes are routed in different contexts when using Calico as a CNI.

-   [Pod to pod communication using an underlay in a single subnet](https://medium.com/@Kubway/communications-between-pods-in-k8s-a4bf633dba03)
-   Pod to pod communication using an underlay with BGP (that part)

[In an other story, we will see how communications between pods and services](https://medium.com/@Kubway/communications-between-pods-and-clusterip-services-in-kubernetes-7cc36c6bbfb9) are managed in a cluster.

For that, we will use TCPDUMP, Wireshark and linux command as traceroute for watching the packets. In the first part, we had a look on the traffic between 2 containers/pods which were located on nodes on the same subnet. On this second part, we will use nodes in different subnet separated by routers and using BGP, with or without VXLAN.

Two cases of communications are studied in this second part:

-   Case 4 : BGP without VXLAN.
-   Case 5 : BGP and VXLAN for tunneling.

We saw in the previous case than traffic between pods is just routed without NAT. So, this new case, even it is more complex, is not so much different. Always routing … Source and destination addresses remains the same.

![[Raw/Media/Resources/318e5b764c2fe18486c9af21bc451bc4_MD5.png]]

**Reconfigure kubeadm2**

Because in this new case, kubeadm2 will not be in the same subnet than kubeadm1, first remove it from the cluster and then change its IP address and gateway. At this time, kubeadm2 can’t access to kubeadm1 because routes are not valid between the 2 nodes.

kubectl delete node kubeadm2  
  
sudo kubeadm reset

**Create the backbone network**

We added 2 routers in between nodes of the cluster. BGP (Border Gateway Protocol) will be used for routing traffic between the pods. Bird is included in Calico and we added the same software Bird on the routers. Notice, we could use an different software or even some physical routers.

**router1 configuration**

These is the configuration files for network on an Ubuntu 24.04 LTS and for bird.

  
network:  
    ethernets:  
        lo:    
          addresses:  
          \- 10.0.0.1/32   
        ens33:   
            addresses:  
            \- 192.168.8.10/24  
            routes:  
            \- to: default  
              via: 192.168.8.2  
            nameservers:  
              addresses:  
              \- 8.8.8.8  
              \- 0.0.0.0  
        ens37:   
            addresses:  
            \- 192.168.16.1/24  
        ens38:   
            addresses:  
            \- 192.168.18.1/24  
    version: 2

\# /etc/bird/bird.conf  
log syslog all;  
router id 10.0.0.1; # loopback address  
protocol device {  
}  
protocol direct {  
  interface "lo", "ens37", "ens38";  
}  
protocol kernel {  
    export all;  
}  
protocol static {  
}  
protocol bgp router2 {  
  local 192.168.18.1 as 64513;  
  neighbor  192.168.18.2 as 64513;  
  next hop self;  
  import all;  
  export all;  
}  
protocol bgp kubeadm1 {  
  local 192.168.16.1 as 64513;  
  neighbor  192.168.16.101 as 64512;  
  import all;  
  export all;  
}

**router2 configuration**

\# /etc/netplan/50-cloud-init.yaml  
network:  
    ethernets:  
        lo:  
          addresses:  
          - 10.0.0.2/32  
        ens33:  
            addresses:  
            - 192.168.18.2/24  
            routes:  
            - to: default  
              via: 192.168.18.1  
        ens37:  
            addresses:  
            - 192.168.17.1/24  
    version: 2

  
log syslog all;  
router id 10.0.0.2;   
protocol device {  
}  
protocol direct {  
}  
protocol kernel {  
    export all;  
}  
protocol static {  
}  
protocol bgp router1 {  
  local 192.168.18.2 as 64513;  
  neighbor 192.168.18.1 as 64513;  
  next hop self;  
  import all;  
  export all;  
}  
protocol bgp kubeadm2 {  
  local 192.168.17.1 as 64513;  
  neighbor  192.168.17.101 as 64512;  
  import all;  
  export all;  
}

At this time, BGP peering is established between routers, so the 2 nodes can access each other and kubeadm2 can be added to the cluster.

**Extending the cluster**

\# From kubeadm1  
kubeadm token create   
\# From kubeadm2  
sudo kubeadm join 192.168.16.101:6443 

kubeadm2 is now part of the cluster :

![[Raw/Media/Resources/d10abc903427bb2e309dce8465a9a791_MD5.png]]

**Modify Calico configuration**

Because in this part, nodes are not anymore in the same subnet and we do not want VXLAN encapsulation in case 4, we have to force that.

  
sudo sed -i 's+encapsulation: VXLANCrossSubnet+encapsulation: None+' custom-resources.yaml

**BGP configuration in the cluster**

First, let’s restart BGP on the cluster we stopped in the first part.

  
kubectl patch installation default --type=merge -p '{"spec": {"calicoNetwork": {"bgp": "Enabled"}}}'

and then configure it

kubectl apply \-f \- << EOF  
apiVersion: projectcalico.org/v3  
kind: BGPConfiguration  
metadata:  
  name: default  
spec:  
  logSeverityScreen: Info  
  nodeToNodeMeshEnabled: false  
  asNumber: 64512   
  serviceClusterIPs:  
  \- cidr: 10.97.0.0/23   
\---  
apiVersion: projectcalico.org/v3  
kind: BGPPeer  
metadata:  
  name: router1  
spec:  
  node: kubeadm1  
  peerIP: 10.0.0.1  
  asNumber: 64513  
\---  
apiVersion: projectcalico.org/v3  
kind: BGPPeer  
metadata:  
  name: router2  
spec:  
  node: kubeadm2  
  peerIP: 10.0.0.2  
  asNumber: 64513  
EOF

**BGP Peering checking**

Right now, we can see the established peering from the routers with birdc, from a node with calicoctl and from calico-node pods with birdcl

**\# From router1 with birdc**  
kubway@router1:~$ sudo birdc show protocols  
BIRD 1.6.8 ready.  
name proto table state since info  
…  
router2 BGP master up 05:40:51 **Established**  
kubeadm1 BGP master up 05:39:15 **Established**

**\# From router2 with birdc**  
kubway@router2:~$ sudo birdc show protocols  
BIRD 1.6.8 ready.  
name proto table state since info  
…  
router1 BGP master up 18:30:14 **Established**  
kubeadm2 BGP master up 18:30:12 **Established**

**\# From kubeadm1 node with calicoctl**  
kubway@kubeadm1:~$ sudo calicoctl node status  
\[sudo\] password for kubway:  
Calico process is running.

IPv4 BGP status  
+--------------+---------------+-------+----------+-------------+  
| PEER ADDRESS |   PEER TYPE   | STATE |  SINCE   |    INFO     |  
+--------------+---------------+-------+----------+-------------+  
| 10.0.0.1     | node specific | up    | 08:33:56 | **Established** |  
+--------------+---------------+-------+----------+-------------+IPv6 BGP status  
No IPv6 peers found.**\# From kubeadm2 node with calicoctl**  
kubway@kubeadm2:~$ sudo calicoctl node status  
Calico process is running.IPv4 BGP status  
+--------------+---------------+-------+----------+-------------+  
| PEER ADDRESS |   PEER TYPE   | STATE |  SINCE   |    INFO     |  
+--------------+---------------+-------+----------+-------------+  
| 10.0.0.2     | node specific | up    | 08:33:57 | **Established** |  
+--------------+---------------+-------+----------+-------------+IPv6 BGP status  
No IPv6 peers found.

**\# From calico-node pod on kubeadm1 with birdcl**  
kubectl exec -it -n calico-system calico-node-r46r7 - birdcl show protocols  
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), install-cni (init)  
BIRD v0.3.3+birdv1.6.8 ready.  
name proto table state since info  
…  
Node\_10\_0\_0\_1 BGP master up 08:33:56 **Established**

**\# From calico-node pod on kubeadm2 with birdcl**  
kubway@kubeadm1:~$ kubectl exec -it -n calico-system calico-node-2bnst -- birdcl show protocols  
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), install-cni (init)  
BIRD v0.3.3+birdv1.6.8 ready.  
name     proto    table    state  since       info  
...  
Node\_10\_0\_0\_2 BGP      master   up     08:33:57    **Established**

**Routing tables on router1 (in the kernel and in bird process)**

You can see learned routes in the Kernel and in the bird process for debugging purposes.

**kubway@router1:~$ ip route** | grep  bird # Routing table in the Kernel  
10.0.0.2 via 192.168.18.2 dev ens38 proto bird  
10.245.0.64/26 via 192.168.16.101 dev ens37 proto bird  
10.245.0.192/26 via 192.168.18.2 dev ens38 proto bird  
192.168.17.0/24 via 192.168.18.2 dev ens38 proto bird

**kubway@router1:~$ sudo birdc show route** # Routing table in the bird process  
BIRD 1.6.8 ready.  
10.0.0.2/32        via 192.168.18.2 on ens38 \[router2 08:32:37\] \* (100/0) \[i\]  
10.0.0.1/32        dev lo \[direct1 08:30:59\] \* (240)  
192.168.16.0/24    dev ens37 \[direct1 08:30:59\] \* (240)  
192.168.17.0/24    via 192.168.18.2 on ens38 \[router2 08:32:37\] \* (100/0) \[i\]  
192.168.18.0/24    dev ens38 \[direct1 08:30:59\] \* (240)  
                   via 192.168.18.2 on ens38 \[router2 08:32:37\] (100/0) \[i\]  
10.245.0.192/26    via 192.168.18.2 on ens38 \[router2 13:25:43\] \* (100/0) \[AS64512i\]  
10.245.0.64/26     via 192.168.16.101 on ens37 \[kubeadm1 13:25:43\] \* (100) \[AS64512i\]

**Routing tables on router2 (in the kernel and in bird process)**

**kubway@router2:~$ ip route | grep  bird**  
10.0.0.1 via 192.168.18.1 dev ens33 proto bird  
10.245.0.64/26 via 192.168.18.1 dev ens33 proto bird  
10.245.0.192/26 via 192.168.17.101 dev ens37 proto bird  
192.168.16.0/24 via 192.168.18.1 dev ens33 proto bird

**kubway@router2:~$** **sudo birdc show route # Routing table in the bird process**  
BIRD 1.6.8 ready.  
10.0.0.2/32        dev lo \[direct1 18:30:10\] \* (240)  
10.0.0.1/32        via 192.168.18.1 on ens33 \[router1 18:30:14\] \* (100/0) \[i\]  
192.168.16.0/24    via 192.168.18.1 on ens33 \[router1 18:30:14\] \* (100/0) \[i\]  
192.168.17.0/24    dev ens37 \[direct1 18:30:10\] \* (240)  
192.168.18.0/24    dev ens33 \[direct1 18:30:10\] \* (240)  
                   via 192.168.18.1 on ens33 \[router1 18:30:14\] (100/0) \[i\]  
10.245.0.192/26    via 192.168.17.101 on ens37 \[kubeadm2 23:23:19\] \* (100) \[AS64512i\]  
10.245.0.64/26     via 192.168.18.1 on ens33 \[router1 23:23:19\] \* (100/0) \[AS64512i\]

**Routing tables in calico-node**

\# Routes in calico-node on kubeadm1 subnet  
kubway@kubeadm1:~$ kubectl exec -it -n calico-system calico-node-r46r7 -- ip route  
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), install-cni (init)  
default via 192.168.16.1 dev ens33 proto static  
10.0.0.1 via 192.168.16.1 dev ens33 proto bird  
10.0.0.2 via 192.168.16.1 dev ens33 proto bird  
10.245.0.64 dev cali56b6dcf9efb scope link  
blackhole 10.245.0.64/26 proto bird  
10.245.0.65 dev calicf4384f6d07 scope link  
10.245.0.66 dev calia52505a6695 scope link  
10.245.0.67 dev calic2e3ce163e0 scope link  
10.245.0.68 dev cali083ecf92347 scope link  
10.245.0.69 dev calibcf2121dbc6 scope link  
10.245.0.70 dev calid74b528052f scope link  
192.168.16.0/24 dev ens33 proto kernel scope link src 192.168.16.101  
192.168.17.0/24 via 192.168.16.1 dev ens33 proto bird  
192.168.18.0/24 via 192.168.16.1 dev ens33 proto bird

\# Routes in calico-node on kubeadm2 subnet  
**kubway@kubeadm1:~$ kubectl exec -it -n calico-system calico-node-2bnst -- ip route**  
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), install-cni (init)  
default via 192.168.17.1 dev ens33 proto static  
10.0.0.1 via 192.168.17.1 dev ens33 proto bird  
10.0.0.2 via 192.168.17.1 dev ens33 proto bird  
10.245.0.192 dev cali2a40b4df543 scope link  
blackhole 10.245.0.192/26 proto bird  
10.245.0.193 dev cali031530bd193 scope link  
192.168.16.0/24 via 192.168.17.1 dev ens33 proto bird  
192.168.17.0/24 dev ens33 proto kernel scope link src 192.168.17.101  
192.168.18.0/24 via 192.168.17.1 dev ens33 proto bird

**Create the pods on the 2 nodes**

See previous part for creating pods in the 2 different nodes.

![[Raw/Media/Resources/3e48784d38e72c7f5eb1746a04726102_MD5.png]]

Capture http traffic on pod-nginx and generate it from pod-alpine to pod-nginx (see previous story).

As you can see, IP source and destination remains unchanged. We could watch the same in pod-alpine, in the nodes or in the routers.

![[Raw/Media/Resources/bd954af6ca3949d26adc515699c2bd33_MD5.png]]

The only difference is the path from pod-alpine to pod-nginx going throught the nodes acting as routers.

![[Raw/Media/Resources/52d44b1c7035f11c43e45872f3bcab42_MD5.png]]

**Modifying Calico configuration**

In this case, we will continue to use BGP but change back encapsulation value field to “VXLANCrossSubnet”. Because nodes are not on the same subnet, encapsulation will happen, change “VXLANCrossSubnet” to “VXLAN” to force encapsulation if you prefer.

  
sudo sed -i 's+encapsulation: None+encapsulation: VXLANCrossSubnet+' custom-resources.yaml

A VXLAN is a virtual layer 2 network (known as the overlay network) built on top of an existing physical layer 3 network (known as the underlay network). VXLAN uses a logical tunnel to transport traffic between two endpoints.

![[Raw/Media/Resources/15bd893bba86bb5e118fc2560e9bf146_MD5.png]]

In this scenario, you are not aware of the network in between the nodes, so you not have to configure routers which is not your responsability. You just have to configure BGPConfiguration and BGPPeers in the cluster with the value your provider gave you.

**Generating traffic between pods**

We still use the same pods than in the case 4.

![[Raw/Media/Resources/01f339a19f79e77204ffb7f7b617a75d_MD5.png]]

Capture http traffic on pod-nginx first and generate it from pod-alpine to pod-nginx. You will not see any difference watching traffic in the pods neither with tcpdump or traceroute .

If you want to see VXLAN encapsulation, you have to capture traffic on the routers which acts as VTEP. A VXLAN tunnel endpoint (VTEP) is a VXLAN-capable device that encapsulates and de-encapsulates packets. Encapsulation happens between VTEP, so between nodes !

Notice than you will not capture the traffic if using pods IP as filter even they stay unchanged, that’s why we will use udp port 4689 instead.

kubway@router1:~$ sudo tcpdump -i ens37 udp and port 4789 -w bgpwithvxlan.pcap

![[Raw/Media/Resources/96f7a6b69a4cd50a9fa72c3f14bee22b_MD5.png]]

-   In the 5 cases we saw than source and destination IPs pods remains the ones of the pods where the containers are. There is only routing and never translation in communications between pods when services are not used.
-   VXLAN is just tunneling in between VXLAN tunnel endpoints which are the nodes.