---
title: Communications between pods and ClusterIP Services in Kubernetes
source: https://medium.com/@Kubway/communications-between-pods-and-clusterip-services-in-kubernetes-7cc36c6bbfb9
clipped: 2025-03-01
published: 
category: network
tags:
  - k8s
read: false
---


**Author : Marie F. from Kubway.**

We saw previously how communications between pods in Kubernetes are routed in different contexts.

-   [Pod to pod communication using an underlay in a single subnet](https://medium.com/@Kubway/communications-between-pods-in-k8s-a4bf633dba03)
-   [Pod to pod communication using](https://medium.com/@Kubway/communications-between-pods-in-k8s-a4bf633dba03) [an underlay with BGP](https://medium.com/@Kubway/communications-between-pods-in-k8s-2a952d409581)

It’s time now to see how communications between pods and services are managed in a cluster, still using Calico as the CNI. We will see only internal communications in the cluster in that story, leaving those with the outside world until next time.

Translation between services to pods can be done in different ways, we will see the 3 major cases in linux systems : Kube-proxy to configure Iptables or IPVS dataplanes and finally eBPF without the need of IPTables.

This story is not for choosing which is the best mode for managing service but getting a basic knowledge of how configure them and how they works. For that, we will use TCPDUMP and Wireshark for watching the packets, iptables, ipvsadm or calico-node -bpf for seeing tables.

Having a basic experience ok Kubernetes,kubectl and Network protocols is necessary for this lab. We will use an existing kubernetes cluster version 1.31 with 2 nodes, using calico as the CNI installed in Case 3 [in this story](https://medium.com/@Kubway/communications-between-pods-in-k8s-a4bf633dba03)

First, we create a deployment with two replicas of image httpd plus one pod with image alpine to access the web-server workload.

k run pod-alpine \--image alpine -- sleep 1d  
k create deploy monwebserver \--replicas 2 \--image httpd

![[Raw/Media/Resources/8124bf6e7c815f76b05131c077067f2d_MD5.png]]

Check the 2 replicas are not on the same node and if not, adjust it. Then, create a service for the deployment and install curl in pod-alpine.

k expose deployment monwebserver   
k exec \-it pod\-alpine   
k exec \-it pod\-alpine 

Service get an IP and 2 endpoints for delivering the service.

![[Raw/Media/Resources/68d4998a0c80e7962532f5819636ce11_MD5.png]]

You can then access the service from the pod “pod-alpine”, there will be a random equal-cost load balancing between the 2 backing pods of the deployment “monwebserver”.

![[Raw/Media/Resources/401f0a5649d56b3dcbee4fdd539c7e0c_MD5.png]]

When there is a request to the service IP (10.97.0.69:8080), the destination is translated to one of the endpoint addresses, this is done by DNAT (Destination NAT) in Iptables.

Iptables is a Linux kernel feature that was designed to be an efficient firewall with sufficient flexibility to handle a wide variety of common packet manipulation and filtering needs. This is the default case used by Kubernetes in Linux systems and it is in that mode that service in previous paragraph is working. In this scenario there is one kube-proxy pod in each node.

![[Raw/Media/Resources/b34fabe3ffa6c023f80a011295867bbc_MD5.png]]

and ”curl -v localhost:10249/proxyMode” tell you “iptables” is on and use the proxy-mode (last word in the return).

You can check the kube-proxy logs too

![[Raw/Media/Resources/780cdec3b049f3e38b68f221293e9223_MD5.png]]

So in case 1, we use nat table of iptables to translate a service address to a pod address. You can see that in this very partial and focused extract you can get with “sudo iptables -t nat -L”. Notice that these entries are on all of the cluster nodes.

![[Raw/Media/Resources/561724b8cda3e2872de59858302a6dcf_MD5.png]]

![[Raw/Media/Resources/7c02f6ac5eeae088330ae9fd5fe55511_MD5.png]]

So, communications from the pod alpine which is running in kubadm1 to the cluster service will be redirect to one of the 2 nginx pods which are running in kubadm1 and kubeadm2. After the translation is done, it is routed to the backend httpd pod.

We will verify that using tcpdump and Wireshark but first identify the interfaces on kubeadm1 we are interresting in with “calicoctl get weps”

![[Raw/Media/Resources/6afca518d0208598cf2f145b5fd84ae2_MD5.png]]

![[Raw/Media/Resources/f88ef50709397fde8d3fa3327f11fb67_MD5.png]]

Generate 10 http request from pod “pod-alpine” to the service and capture packets in the 3 places.

for i in {1..10}; do  k exec -it pod-alpine -- curl svc-webserver:8080; done

**Capture 1** : This one is the traffic before DNAT, destination is the IP address of the service. All the 10 requests are captured.

sudo tcpdump -i cali36f416bcac8 src 10.245.0.73 -w cali36f416bcac8.pcap

![[Raw/Media/Resources/af83cf1dcfa7c885208e9a99c1e67949_MD5.png]]

**Capture 2**: It is around half of the traffic coming from pod-alpine after DNAT which is going that way. The new destination IP address is the one of the pod httpd on kubeadm2, source address didn’t change. Moreover, this traffic is not VXLAN encapsulated because source and destination nodes are on the same subnet (“VXLANCrossSubnet” has been used in calico configuration).

sudo tcpdump -i ens33 src 10.245.0.73 -w ens33.pcap

![[Raw/Media/Resources/97890f2a5a62776e1709b6c56bf12b3c_MD5.png]]

**Capture 3**: This is the second half of the traffic coming from pod-alpine after DNAT. The new destination address is the one of the pod nginx on kubeadm1. This traffic is not VXLAN encapsulated.

sudo tcpdump -i cali6857ce3f6ad src 10.245.0.73 -w cali6857ce3f6ad.pcap

![[Raw/Media/Resources/bda7a2679b80d72ea5c0eac835724e4a_MD5.png]]

IPVS is an other kernel feature designed specifically for load balancing. In this mode, Kube-Proxy inserts rules into IPVS instead of IPtables.

**Prepare the Kubernetes nodes for IPVS mode**

  
sudo modprobe -- ip\_vs  
sudo modprobe -- ip\_vs\_rr  
sudo modprobe -- ip\_vs\_wrr  
sudo modprobe -- ip\_vs\_sh  
sudo modprobe -- nf\_conntrack\_ipv4  
  
sudo sh -c 'cat << EOF > /etc/modules-load.d/ipvs.conf  
ip\_vs  
ip\_vs\_rr  
ip\_vs\_wrr  
ip\_vs\_sh  
nf\_conntrack  
EOF'  
  
sudo apt-get install ipvsadm -y

**Change Kube-Proxy mode from “iptables” to “ipvs”**

kubectl edit configmap kube-proxy -n kube-system  

  
k rollout restart ds -n kube-system kube-proxy

Verify the new mode

![[Raw/Media/Resources/528b8ad40d44012332fdde7605c0b4ab_MD5.png]]

Create a new service and get its IP

k create deploy monwebserver2 \--replicas 2 \--image nginx  
k expose deployment monwebserver2 \--name svc-webserver2 \--port 8080 \--target-port 80

![[Raw/Media/Resources/d80688e6c6652a3f5bbbc1e013213904_MD5.png]]

You can see that both services (the one which has been created in iptables mode and the new one created in ipvs mode) are in the ipvs translation table. Moreover, you can verify that both services are working properly with curl.

![[Raw/Media/Resources/126606b288d2f6d3c7ecd9dd20b6cafe_MD5.png]]

In Iptables which is not used anymore by Kube-proxy, but still for other services, Kubernetes Services are not any more in KUBE-SERVICES chain. You can notice than even they are not used anymore for the previous reason, DNAT entries are still in iptables.

![[Raw/Media/Resources/377658b22300d2597d59b73af106c599_MD5.png]]

For my curiosity, I revert to iptables mode and watch for the 2 rules, I saw the 2 services were still managed. I don’t know if it is garanteed in all cases …

![[Raw/Media/Resources/38f3ec002852c989310d1fea05e1d659_MD5.png]]

Calico is a control plane that programs Iptables or IPVS dataplanes but also eBPF (Extended Berkeley Packet Filter), Windows HNS (Host Network Service) and even VPP from Cisco which are alternatives to iptables with kube-proxy.

If you are using Calico as a CNI in Kubernetes with Iptables and want to switch to eBPF, migrating to Cilium can be an option. Staying with Calico can be a easier if you want to migrate smoothly when you already have a production environment that works well with Calico, especially if you value simplicity, maturity, and network management based on proven standards like BGP.

eBPF is a technology that can run sandboxed programs in the operating system kernel. It is used to safely and efficiently extend the capabilities of the kernel without requiring to change kernel source code or load kernel modules. Calico offers eBPF data plane as an alternative to the standard Linux iptables dataplane with several advantages :

-   It scales to higher throughput and reduces latency
-   It uses less CPU than kube-proxy to keep the dataplane in sync.
-   It has native support for Kubernetes services (without needing kube-proxy).
-   It preserves external client source IP addresses all the way to the pod.
-   It supports DSR (Direct Server Return) for more efficient service routing.

**Rebuild the lab**

I get some problems migrating from Kube-Proxy/IPVS to eBPF, I didn’t find any informations about this migration. So, I reconstructed the lab and that’s why IPs are differents.

Let’smigrate from Kube-Proxy/IPTables to eBPF, service svc-webserver is ready at this time.

![[Raw/Media/Resources/03677e88abf26f5981f829e60457c687_MD5.png]]

**Enable eBPF**

Calico-nodes will not not have anymore to talk to the proxy but instead directly with kube-apiserver. So the first step is to declare it in a config map.

kubectl apply -f - << EOF  
kind: ConfigMap  
apiVersion: v1  
metadata:  
  name: kubernetes-services-endpoint  
  namespace: tigera-operator  
data:  
  KUBERNETES\_SERVICE\_HOST:   
  KUBERNETES\_SERVICE\_PORT:   
EOF

Then, instead of removing kube-proxy definitively, modify nodeSelector value in its daemonset with a value that nodes don’t have as a label.

  
kubectl get ds -n kube-system kube-proxy -o=jsonpath='{.spec.template.spec.nodeSelector}'  
  
kubectl patch ds -n kube-system kube-proxy -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": "true"}}}}}'

After modified nodeSelector, you can check than there is no more kube-proxy pods. Howewer, pods which were running are still alive, iptables tables stay unchanged !

![[Raw/Media/Resources/6a092599e797c91c9d2bb1cdad78d7e1_MD5.png]]

Switch from Iptables to eBPF

\# Check which dataplane is active  
k get  installation default -o=jsonpath='{.spec.calicoNetwork.linuxDataplane}'  
\# Enable eBPF  
kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"linuxDataplane":"BPF"}}}'

**Observe tables**

A few minutes later, when calico pods has been restarted, you will notice than Iptables still exist but has been modified. Especially, you can notice that there is not anymore entries with DNAT

![[Raw/Media/Resources/2cedbd412c9a5f308c1011f847ad991e_MD5.png]]

Howewer nat from service to pods still working

![[Raw/Media/Resources/6628ad9c5cd41c8a07c3615f9adbe1a6_MD5.png]]

In a calico-node pod, you can use calico-node utility to show or sometimes set entries

![[Raw/Media/Resources/cea46938db71569e705b6e573aafc99a_MD5.png]]

So, we can show the nat entries with “calico-node -bpf nat dump” execute in calico-node, the one running in kubeadm1 for example.

![[Raw/Media/Resources/7ae0e7323ef14629e871cdc5e6722ef4_MD5.png]]

You can see connexions too. For example, just after a request from pod-alpine to the service svc-webserver.

![[Raw/Media/Resources/936b96ca64e006cd6ed3988513c31dd4_MD5.png]]

**Reversing to Kube-proxy mode**

  
kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"linuxDataplane":"Iptables"}}}'  
  
kubectl patch ds -n kube-system kube-proxy --type merge -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": null}}}}}'

[https://ebpf.io/what-is-ebpf/](https://ebpf.io/what-is-ebpf/)

[https://docs.tigera.io/calico/latest/operations/ebpf/enabling-ebpf](https://docs.tigera.io/calico/latest/operations/ebpf/enabling-ebpf)

[https://www.youtube.com/watch?v=Mh43sNBu208](https://www.youtube.com/watch?v=Mh43sNBu208)

If the traffic from a pod to another pod is just routed, traffic from a pod to a service is natted, the destination address is changed from service address to one of the endpoint (a pod that serves the service).

This can be done by tables in IPTables, IPVS or eBPF.