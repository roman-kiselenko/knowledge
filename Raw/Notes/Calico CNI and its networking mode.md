---
title: Calico CNI and its networking mode
source: https://sigridjin.medium.com/calico-cni-and-its-networking-mode-ff71c7bdf65b
clipped: 2025-02-16
published: 
category: k8s
tags:
  - network
read: false
---

We delved deep into Calico, a cornerstone of Kubernetes networking. As we progress through our studies, building knowledge piece by piece can be challenging, but it’s incredibly rewarding to learn new concepts and technologies. We focused on Calico, a Container Network Interface (CNI) for Kubernetes, which was briefly mentioned alongside Flannel CNI in our previous session.

1.  Pod-to-Pod communication without NAT
2.  Node-to-Pod communication
3.  Host-mode Pods must communicate with other Pods without NAT
4.  Service IP ranges must not overlap with Pod IP ranges

These requirements form the foundation of any CNI, including popular options like EKS VPC CNI, Calico, and Flannel. While each CNI has its unique features and operational methods, they all must adhere to these interface standards.

![[Raw/Media/Resources/112cf9104fa2f0d0dd79ba55271b8a44_MD5.png]]

Calico is one of the most widely adopted CNIs, offering a broad range of networking modes to suit various requirements. Its versatility extends beyond Kubernetes, making it suitable for diverse environments. According to Calico’s official documentation, it’s compatible with nearly all infrastructure setups, including:

-   On-premises data centers
-   Public clouds (AWS, GCP, Azure)
-   OpenStack environments
-   Bare metal servers
-   Virtual machines

One interesting discovery from the official documentation is that Calico offers multiple product tiers, each with distinct feature sets. Also, it leverages BGP for efficient routing, allowing for scalable and performant networking in large clusters. For a detailed comparison, refer to the official Calico documentation at [https://docs.tigera.io/](https://docs.tigera.io/).

Calico’s architecture consists of several key components.

1.  Bird: An open-source BGP routing daemon that distributes routes from Felix to BGP peers (nodes) in the network, facilitating inter-host routing.
2.  Felix: A daemon that manages routes, ACLs, and other configurations on each host to provide desired connectivity for endpoints.
3.  Confd: A lightweight, open-source configuration management tool that monitors changes in the Calico Datastore for BGP configurations, AS numbers, logging levels, and IPAM information.
4.  IPAM (IP Address Management): Controls IP address allocation to Pods within the cluster using Calico’s IP pool resources.
5.  Typha: Acts as a cache between the datastore and Felix instances to improve scalability by reducing the load on the datastore.
6.  Datastore: Stores Calico-related configurations, similar to Kubernetes’ etcd. It supports both Kubernetes API and etcd storage methods, with the API method being simpler to manage as it doesn’t require an additional datastore.

![[Raw/Media/Resources/f8c5bd0abbb6e7dd5c6b13ce110f375e_MD5.png]]

Calico supports several networking modes, each with its own characteristics:

1.  IPIP Mode: Similar to Flannel, encapsulating packets for inter-node communication.
2.  Direct Mode: Packets are routed directly between nodes based on routing information, without encapsulation.
3.  VXLAN Mode: Inter-node Pod communication occurs through VXLAN interfaces, encapsulating L2 frames in UDP-VXLAN packets.
4.  Pod Packet Encryption Mode: Utilizes WireGuard tunnels to automatically encrypt and securely transmit Pod traffic between nodes.

## Practice: Initial Setup

The instructions begin with setting up a Kubernetes cluster using AWS CloudFormation. This creates a multi-node cluster with a master node (k8s-m) and worker nodes.

![[Raw/Media/Resources/c3502efb89d5a1fb41272da8fe26e46b_MD5.png]]

curl -O https://s3.ap-northeast-2.amazonaws.com/cloudformation.cloudneta.net/kans/kans-3w.yaml

aws cloudformation deploy --template-file kans-3w.yaml --stack-name mylab --parameter-overrides KeyName=kp-gasida SgIngressSshCidr=$(curl -s ipinfo.io/ip)/32 --region ap-northeast-2

aws cloudformation deploy --template-file kans-3w.yaml --stack-name mylab --parameter-overrides MyInstanceType=t2.micro KeyName=kp-gasida SgIngressSshCidr=$(curl -s ipinfo.io/ip)/32 --region ap-northeast-2  
aws cloudformation describe-stacks --stack-name mylab --query 'Stacks\[\*\].Outputs\[0\].OutputValue' --output text --region ap-northeast-2

while true; do   
  date  
  AWS\_PAGER="" aws cloudformation list-stacks \\  
    --stack-status-filter CREATE\_IN\_PROGRESS CREATE\_COMPLETE CREATE\_FAILED DELETE\_IN\_PROGRESS DELETE\_FAILED \\  
    --query "StackSummaries\[\*\].{StackName:StackName, StackStatus:StackStatus}" \\  
    --output table  
  sleep 1  
done

ssh -i ~/.ssh/kp-test.pem ubuntu@$(aws cloudformation describe-stacks --stack-name mylab --query 'Stacks\[\*\].Outputs\[0\].OutputValue' --output text --region ap-northeast-2)

Before installing Calico, the network configuration is minimal:

-   The `ip addr` output shows only the loopback (lo) and the primary network interface (ens5).
-   IPTables rules are relatively simple, with basic Kubernetes-related chains but no CNI-specific rules.
-   There are 50 filter table rules and 48 NAT table rules.

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# ip \-c addr  
1: lo: <LOOPBACK,UP,LOWER\_UP\> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
    inet 127.0.0.1/8 scope host lo  
       valid\_lft forever preferred\_lft forever  
    inet6 ::1/128 scope host  
       valid\_lft forever preferred\_lft forever  
2: ens5: <BROADCAST,MULTICAST,UP,LOWER\_UP\> mtu 9001 qdisc mq state UP group default qlen 1000  
    link/ether 02:47:bd:e3:55:01 brd ff:ff:ff:ff:ff:ff  
    altname enp0s5  
    inet 192.168.10.10/24 metric 100 brd 192.168.10.255 scope global dynamic ens5  
       valid\_lft 3328sec preferred\_lft 3328sec  
    inet6 fe80::47:bdff:fee3:5501/64 scope link  
       valid\_lft forever preferred\_lft forever  
(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~#

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# iptables \-t filter \-L  
Chain INPUT (policy ACCEPT)  
target     prot opt source               destination  
KUBE\-PROXY\-FIREWALL  all    
KUBE\-NODEPORTS  all    
KUBE\-EXTERNAL\-SERVICES  all    
KUBE\-FIREWALL  all  

Chain FORWARD (policy ACCEPT)  
target     prot opt source               destination  
KUBE\-PROXY\-FIREWALL  all    
KUBE\-FORWARD  all    
KUBE\-SERVICES  all    
KUBE\-EXTERNAL\-SERVICES  all  

Chain OUTPUT (policy ACCEPT)  
target     prot opt source               destination  
KUBE\-PROXY\-FIREWALL  all    
KUBE\-SERVICES  all    
KUBE\-FIREWALL  all  

Chain KUBE\-EXTERNAL\-SERVICES (2 references)  
target     prot opt source               destination

Chain KUBE\-FIREWALL (2 references)  
target     prot opt source               destination  
DROP       all  

Chain KUBE\-FORWARD (1 references)  
target     prot opt source               destination  
DROP       all    
ACCEPT     all    
ACCEPT     all  

Chain KUBE\-KUBELET\-CANARY (0 references)  
target     prot opt source               destination

Chain KUBE\-NODEPORTS (1 references)  
target     prot opt source               destination

Chain KUBE\-PROXY\-CANARY (0 references)  
target     prot opt source               destination

Chain KUBE\-PROXY\-FIREWALL (3 references)  
target     prot opt source               destination

Chain KUBE\-SERVICES (2 references)  
target     prot opt source               destination  
REJECT     tcp    
REJECT     tcp    
REJECT     udp    
(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~#

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# iptables \-t nat \-L  
Chain PREROUTING (policy ACCEPT)  
target     prot opt source               destination  
KUBE\-SERVICES  all  

Chain INPUT (policy ACCEPT)  
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)  
target     prot opt source               destination  
KUBE\-SERVICES  all  

Chain POSTROUTING (policy ACCEPT)  
target     prot opt source               destination  
KUBE\-POSTROUTING  all  

Chain KUBE\-KUBELET\-CANARY (0 references)  
target     prot opt source               destination

Chain KUBE\-MARK\-MASQ (2 references)  
target     prot opt source               destination  
MARK       all  

Chain KUBE\-NODEPORTS (1 references)  
target     prot opt source               destination

Chain KUBE\-POSTROUTING (1 references)  
target     prot opt source               destination  
RETURN     all    
MARK       all    
MASQUERADE  all  

Chain KUBE\-PROXY\-CANARY (0 references)  
target     prot opt source               destination

Chain KUBE\-SEP\-FKILGDVKESVSEHEB (1 references)  
target     prot opt source               destination  
KUBE\-MARK\-MASQ  all    
DNAT       tcp  

Chain KUBE\-SERVICES (2 references)  
target     prot opt source               destination  
KUBE\-SVC\-NPX46M4PTMTKRN6Y  tcp    
KUBE\-NODEPORTS  all  

Chain KUBE\-SVC\-NPX46M4PTMTKRN6Y (1 references)  
target     prot opt source               destination  
KUBE\-MARK\-MASQ  tcp    
KUBE\-SEP\-FKILGDVKESVSEHEB  all    
(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~#

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# iptables \-t nat \-L | wc \-l  
48  
(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# iptables \-t filter \-L | wc \-l  
50

Calico is installed using a custom YAML file.

-   Calico daemon sets for each node
-   Custom Resource Definitions (CRDs) for Calico’s components
-   RBAC rules for Calico’s operation
-   ConfigMaps for Calico configuration

After installing Calico, several changes occur.

a. New Pods:

-   Calico controller pod is deployed
-   Calico node pods are deployed on each Kubernetes node
-   CoreDNS pods transition from Pending to Running state

b. Networking interfaces.

-   New virtual interfaces (like cali\* and tunl0) should appear
-   Routing tables would be updated to include routes for pod networks
-   IPTables rules would significantly increase to handle pod-to-pod and pod-to-service communication

c. Functionality:

-   CoreDNS becomes operational, indicating that the cluster’s internal DNS is now functional
-   Pod-to-pod communication across nodes becomes possible

(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~  
poddisruptionbudget.policy/calico-kube-controllers created  
serviceaccount/calico-kube-controllers created  
serviceaccount/calico-node created  
serviceaccount/calico-cni-plugin created  
configmap/calico-config created  
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created  
customresourcedefinition.apiextensions.k8s.io/bgpfilters.crd.projectcalico.org created  
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created  
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created  
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created  
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created  
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created  
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created  
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created  
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created  
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created  
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created  
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created  
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created  
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created  
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created  
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created  
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created  
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created  
clusterrole.rbac.authorization.k8s.io/calico-node created  
clusterrole.rbac.authorization.k8s.io/calico-cni-plugin created  
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created  
clusterrolebinding.rbac.authorization.k8s.io/calico-node created  
clusterrolebinding.rbac.authorization.k8s.io/calico-cni-plugin created  
daemonset.apps/calico-node created  
deployment.apps/calico-kube-controllers created

Please note that it operates as a DaemonSet, ensuring it runs on every node

-   It sets up the necessary networking components for inter-pod and inter-node communication
-   The installation of Calico resolves the ‘Pending’ state of core components like CoreDNS.

(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# kubectl get pod -A -owide  
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE     IP               NODE     NOMINATED NODE   READINESS GATES  
kube-system   calico-kube-controllers-77d59654f4-xgt7f   1/1     Running   0          39s     172.16.158.2     k8s-w1   <none\>           <none\>  
kube-system   calico-node-6pjnv                          1/1     Running   0          39s     192.168.10.101   k8s-w1   <none\>           <none\>  
kube-system   calico-node-gkmrh                          1/1     Running   0          39s     192.168.10.102   k8s-w2   <none\>           <none\>  
kube-system   calico-node-vdxld                          1/1     Running   0          39s     192.168.10.10    k8s-m    <none\>           <none\>  
kube-system   calico-node-whxqz                          1/1     Running   0          39s     192.168.20.100   k8s-w0   <none\>           <none\>  
kube-system   coredns-55cb58b774-djmlk                   1/1     Running   0          5m6s    172.16.158.3     k8s-w1   <none\>           <none\>  
kube-system   coredns-55cb58b774-z482n                   1/1     Running   0          5m6s    172.16.158.1     k8s-w1   <none\>           <none\>  
kube-system   etcd-k8s-m                                 1/1     Running   0          5m21s   192.168.10.10    k8s-m    <none\>           <none\>  
kube-system   kube-apiserver-k8s-m                       1/1     Running   0          5m21s   192.168.10.10    k8s-m    <none\>           <none\>  
kube-system   kube-controller-manager-k8s-m              1/1     Running   0          5m21s   192.168.10.10    k8s-m    <none\>           <none\>  
kube-system   kube-proxy-8s5ws                           1/1     Running   0          5m2s    192.168.10.101   k8s-w1   <none\>           <none\>  
kube-system   kube-proxy-jp6h8                           1/1     Running   0          5m4s    192.168.10.102   k8s-w2   <none\>           <none\>  
kube-system   kube-proxy-mmk4s                           1/1     Running   0          5m5s    192.168.20.100   k8s-w0   <none\>           <none\>  
kube-system   kube-proxy-mmxpz                           1/1     Running   0          5m7s    192.168.10.10    k8s-m    <none\>           <none\>  
kube-system   kube-scheduler-k8s-m                       1/1     Running   0          5m21s   192.168.10.10    k8s-m    <none\>           <none\>

The Calico CNI is installed using a custom YAML file, and it contains all necessary components for Calico, including DaemonSets, ConfigMaps, and Custom Resource Definitions (CRDs). When creating new CRDs, then new virtual interfaces are created, including `tunl0` for IPIP tunneling; the `cali*` interfaces are created for each pod.

Also New routes are added for pod networks, utilizing the `tunl0` interface; Approximately 60 new rules are added to both the filter and NAT tables, which manage pod-to-pod and pod-to-service communication.

(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# curl -L https://github.com/projectcalico/calico/releases/download/v3.28.1/calicoctl-linux-amd64 -o calicoctl  
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current  
                                 Dload  Upload   Total   Spent    Left  Speed  
  0     0    0     0    0     0      0      0 \--:\--:-- \--:\--:-- \--:\--:--     0  
100 64.4M  100 64.4M    0     0  33.5M      0  0:00:01  0:00:01 \--:\--:-- 89.3M  
(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# chmod +x calicoctl  
(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# sudo mv calicoctl /usr/local/bin/  
(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# calicoctl  
Usage:  
  calicoctl \[options\] <command> \[<args>...\]  
Invalid option: ''. Use flag '--help' to read about a specific subcommand.

(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# calicoctl version  
Client Version:    v3.28.1  
Git commit:        601856343  
Cluster Version:   v3.28.1  
Cluster Type:      k8s,bgp,kubeadm,kdd

-   `calicoctl ipam show`: Displays IPAM information, including used and available IPs.
-   `calicoctl node status`: Shows node status and BGP peer information.
-   `calicoctl get ippool`: Retrieves IP pool configurations.
-   `calicoctl get workloadEndpoint`: Lists all pod network endpoints.

(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# tree /opt/cni/bin  
/opt/cni/bin  
├── bandwidth  
├── bridge  
├── calico  
├── calico-ipam  
├── dhcp  
├── dummy  
├── firewall  
├── flannel  
├── host-device  
├── host-local  
├── ipvlan  
├── loopback  
├── macvlan  
├── portmap  
├── ptp  
├── sbr  
├── static  
├── tap  
├── tuning  
├── vlan  
└── vrf

0 directories, 21 files

(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# ls -al /opt/cni/bin  
total 209428  
drwxr-xr-x 2 root root     4096 Sep 22 01:52 .  
drwxr-xr-x 3 root root     4096 Sep 22 01:46 ..  
\-rwxr-xr-x 1 root root  4141145 Sep 22 01:52 bandwidth  
\-rwxr-xr-x 1 root root  4652757 Jan 11  2024 bridge  
\-rwxr-xr-x 1 root root 65288440 Sep 22 01:52 calico  
\-rwxr-xr-x 1 root root 65288440 Sep 22 01:52 calico-ipam  
\-rwxr-xr-x 1 root root 11050013 Jan 11  2024 dhcp  
\-rwxr-xr-x 1 root root  4297556 Jan 11  2024 dummy  
\-rwxr-xr-x 1 root root  4736299 Jan 11  2024 firewall  
\-rwxr-xr-x 1 root root  2586700 Sep 22 01:52 flannel  
\-rwxr-xr-x 1 root root  4191837 Jan 11  2024 host-device  
\-rwxr-xr-x 1 root root  3637994 Sep 22 01:52 host-local  
\-rwxr-xr-x 1 root root  4315686 Jan 11  2024 ipvlan  
\-rwxr-xr-x 1 root root  3705893 Sep 22 01:52 loopback  
\-rwxr-xr-x 1 root root  4349395 Jan 11  2024 macvlan  
\-rwxr-xr-x 1 root root  4178588 Sep 22 01:52 portmap  
\-rwxr-xr-x 1 root root  4470977 Jan 11  2024 ptp  
\-rwxr-xr-x 1 root root  3851218 Jan 11  2024 sbr  
\-rwxr-xr-x 1 root root  3110828 Jan 11  2024 static  
\-rwxr-xr-x 1 root root  4371897 Jan 11  2024 tap  
\-rwxr-xr-x 1 root root  3861557 Sep 22 01:52 tuning  
\-rwxr-xr-x 1 root root  4310173 Jan 11  2024 vlan  
\-rwxr-xr-x 1 root root  4001842 Jan 11  2024 vrf

As you can see above, Calico uses BGP (Border Gateway Protocol) for routing between nodes, while each node advertises its pod CIDR range to other nodes. Calico manages a large CIDR (e.g., 172.16.0.0/16) and allocates smaller blocks (e.g., /24) to each node.

Calico uses the Bird routing daemon for BGP implementation. The `bird.cfg` file shows BGP peer configurations for inter-node communication. Pod and Service CIDRs can be verified using this command.

kubectl cluster-info dump | grep -m 2 -E "cluster-cidr|service-cluster-ip-range"  
kubectl get cm -n kube-system kubeadm-config -oyaml | grep -i subnet

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# ip \-c route  
default via 192.168.10.1 dev ens5 proto dhcp src 192.168.10.10 metric 100  
172.16.34.0/24 via 192.168.20.100 dev tunl0 proto bird onlink  
blackhole 172.16.116.0/24 proto bird  
172.16.158.0/24 via 192.168.10.101 dev tunl0 proto bird onlink  
172.16.184.0/24 via 192.168.10.102 dev tunl0 proto bird onlink  
192.168.0.2 via 192.168.10.1 dev ens5 proto dhcp src 192.168.10.10 metric 100  
192.168.10.0/24 dev ens5 proto kernel scope link src 192.168.10.10 metric 100  
192.168.10.1 dev ens5 proto dhcp scope link src 192.168.10.10 metric 100

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# ip \-c addr  
1: lo: <LOOPBACK,UP,LOWER\_UP\> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
    inet 127.0.0.1/8 scope host lo  
       valid\_lft forever preferred\_lft forever  
    inet6 ::1/128 scope host  
       valid\_lft forever preferred\_lft forever  
2: ens5: <BROADCAST,MULTICAST,UP,LOWER\_UP\> mtu 9001 qdisc mq state UP group default qlen 1000  
    link/ether 02:47:bd:e3:55:01 brd ff:ff:ff:ff:ff:ff  
    altname enp0s5  
    inet 192.168.10.10/24 metric 100 brd 192.168.10.255 scope global dynamic ens5  
       valid\_lft 2978sec preferred\_lft 2978sec  
    inet6 fe80::47:bdff:fee3:5501/64 scope link  
       valid\_lft forever preferred\_lft forever  
3: tunl0@NONE: <NOARP,UP,LOWER\_UP\> mtu 8981 qdisc noqueue state UNKNOWN group default qlen 1000  
    link/ipip 0.0.0.0 brd 0.0.0.0  
    inet 172.16.116.0/32 scope global tunl0  
       valid\_lft forever preferred\_lft forever

After installing Calico, a series of IPTables rules are added to your nodes to manage network traffic according to Calico’s policies. These rules are part of Calico’s way of enforcing network policies and managing traffic flow between Pods and external networks.

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# iptables \-t filter \-L | grep cali  
cali\-INPUT  all    
cali\-FORWARD  all    
ACCEPT     all    
MARK       all    
cali\-OUTPUT  all    
Chain cali\-FORWARD (1 references)  
MARK       all    
cali\-from\-hep\-forward  all    
cali\-from\-wl\-dispatch  all    
cali\-to\-wl\-dispatch  all    
cali\-to\-hep\-forward  all    
cali\-cidr\-block  all    
Chain cali\-INPUT (1 references)  
ACCEPT     ipencap  
DROP       ipencap  
cali\-wl\-to\-host  all    
ACCEPT     all    
MARK       all    
cali\-from\-host\-endpoint  all    
ACCEPT     all    
Chain cali\-OUTPUT (1 references)  
ACCEPT     all    
RETURN     all    
ACCEPT     ipencap  
MARK       all    
cali\-to\-host\-endpoint  all    
ACCEPT     all    
Chain cali\-cidr\-block (1 references)  
Chain cali\-from\-hep\-forward (1 references)  
Chain cali\-from\-host\-endpoint (1 references)  
Chain cali\-from\-wl\-dispatch (2 references)  
DROP       all    
Chain cali\-to\-hep\-forward (1 references)  
Chain cali\-to\-host\-endpoint (1 references)  
Chain cali\-to\-wl\-dispatch (1 references)  
DROP       all    
Chain cali\-wl\-to\-host (1 references)  
cali\-from\-wl\-dispatch  all    
ACCEPT     all  

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# iptables \-t nat \-L | grep cali  
cali\-PREROUTING  all    
cali\-OUTPUT  all    
cali\-POSTROUTING  all    
Chain cali\-OUTPUT (1 references)  
cali\-fip\-dnat  all    
Chain cali\-POSTROUTING (1 references)  
cali\-fip\-snat  all    
cali\-nat\-outgoing  all    
MASQUERADE  all    
Chain cali\-PREROUTING (1 references)  
cali\-fip\-dnat  all    
Chain cali\-fip\-dnat (2 references)  
Chain cali\-fip\-snat (1 references)  
Chain cali\-nat\-outgoing (1 references)  
MASQUERADE  all  

By counting the total number of rules, we can see the extent of Calico’s integration: Approximately 60 rules are added, demonstrating Calico’s comprehensive control over network traffic.

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# iptables \-t nat \-L | wc \-l  
126  
(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# iptables \-t filter \-L | wc \-l  
108  
(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~#

Calico’s IPAM module handles the allocation of IP addresses to Pods across the cluster. This output shows the total number of IP addresses in the pool, how many are in use, and how many are free. Each `/24` block is assigned to a node and widely known as BGP using the bird protocol.

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# # 칼리코 IPAM 정보 확인 : 칼리코 CNI 를 사용 한 파드가 생성된 노드에 podCIDR 네트워크 대역 확인 \- 링크  
calicoctl ipam show  
+  
| GROUPING |     CIDR      | IPS TOTAL | IPS IN USE |   IPS FREE   |  
+  
| IP Pool  | 172.16.0.0/16 |     65536 | 7 (0%)     | 65529 (100%) |  
+

calicoctl ipam show   
calicoctl ipam show   
calicoctl ipam show 

\# 칼리코 IPAM 정보 확인 : 칼리코 CNI 를 사용한 파드가 생성 된 노드에 podCIDR 네트워크 대역 확인 \- 링크  
calicoctl ipam show

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# # 칼리코 IPAM 정보 확인 : 칼리코 CNI 를 사용 한 파드가 생성된 노드에 podCIDR 네트워크 대역 확인 \- 링크  
calicoctl ipam show  
+  
| GROUPING |     CIDR      | IPS TOTAL | IPS IN USE |   IPS FREE   |  
+  
| IP Pool  | 172.16.0.0/16 |     65536 | 7 (0%)     | 65529 (100%) |  
+  
(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# # Block 는 각 노드에 할당된 podCIDR 정보  
calicoctl ipam show   
calicoctl ipam show   
calicoctl ipam show   
+  
| GROUPING |      CIDR       | IPS TOTAL | IPS IN USE |   IPS FREE   |  
+  
| IP Pool  | 172.16.0.0/16   |     65536 | 7 (0%)     | 65529 (100%) |  
| Block    | 172.16.116.0/24 |       256 | 1 (0%)     | 255 (100%)   |  
| Block    | 172.16.158.0/24 |       256 | 4 (2%)     | 252 (98%)    |  
| Block    | 172.16.184.0/24 |       256 | 1 (0%)     | 255 (100%)   |  
| Block    | 172.16.34.0/24  |       256 | 1 (0%)     | 255 (100%)   |  
+  
+  
| IP | BORROWING\-NODE | BLOCK | BLOCK OWNER | TYPE | ALLOCATED\-TO |  
+  
+  
+  
|      PROPERTY      | VALUE |  
+  
| StrictAffinity     | false |  
| AutoAllocateBlocks | true  |  
| MaxBlocksPerHost   |     0 |  
+

Calico uses BGP (Border Gateway Protocol) for propagating routing information between nodes. The “Peer Address” column lists the IP addresses of the nodes in the mesh. The BGP sessions are established, indicating successful routing configurations.

(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# calicoctl node status  
Calico process is running.

IPv4 BGP status  
+  
|  PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |  
+  
| 192.168.20.100 | node-to-node mesh | up    | 16:52:54 | Established |  
| 192.168.10.101 | node-to-node mesh | up    | 16:52:52 | Established |  
| 192.168.10.102 | node-to-node mesh | up    | 16:52:54 | Established |  
+

IPv6 BGP status  
No IPv6 peers found.

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# calicoctl node checksystem  
Checking kernel version...  
  6.5.0\-1024\-aws           OK  
Checking kernel modules...  
  ip\_set                   OK  
  ip6\_tables               OK  
  ipt\_ipvs                 OK  
  xt\_bpf                   OK  
  ipt\_REJECT               OK  
  xt\_rpfilter              OK  
  xt\_icmp                  OK  
  ipt\_rpfilter             OK  
  ipt\_set                  OK  
  xt\_addrtype              OK  
  xt\_conntrack             OK  
  xt\_multiport             OK  
  xt\_u32                   OK  
  ip\_tables                OK  
  vfio\-pci                 OK  
  nf\_conntrack\_netlink     OK  
  xt\_icmp6                 OK  
  xt\_mark                  OK  
  xt\_set                   OK  
System meets minimum system requirements to run Calico!

Displaying IP Pools:

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# calicoctl get ippool \-o wide  
NAME                  CIDR            NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR  
default\-ipv4\-ippool   172.16.0.0/16   true   Always     Never       false      false              all()

Retrieve the Pod and Service CIDR ranges:

(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~  
kubectl get cm -n kube-system kubeadm-config -oyaml | grep -i subnet  
                            "--service-cluster-ip-range=10.200.1.0/24",  
                            "--cluster-cidr=172.16.0.0/16",  
      podSubnet: 172.16.0.0/16  
      serviceSubnet: 10.200.1.0/24

Workload endpoints represent the network interfaces of Pods. This information includes the namespace, workload name (Pod name), node, assigned IP addresses, and network interfaces.

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# calicoctl get workloadEndpoint  
calicoctl get workloadEndpoint \-A  
calicoctl get workloadEndpoint \-o wide \-A  
WORKLOAD   NODE   NETWORKS   INTERFACE

NAMESPACE     WORKLOAD                                   NODE     NETWORKS          INTERFACE  
kube\-system   calico\-kube\-controllers\-77d59654f4\-xgt7f   k8s\-w1   172.16.158.2/32   cali1a6c68cfc2f  
kube\-system   coredns\-55cb58b774\-djmlk                   k8s\-w1   172.16.158.3/32   cali7760be1255d  
kube\-system   coredns\-55cb58b774\-z482n                   k8s\-w1   172.16.158.1/32   cali1429ee5e74b

NAMESPACE     NAME                                                            WORKLOAD                                   NODE     NETWORKS          INTERFACE         PROFILES                                                  NATS  
kube\-system   k8s  
kube\-system   k8s  
kube\-system   k8s

Calico uses the bird Internet Routing Daemon for BGP route distribution. The `calico-node` DaemonSet runs bird, and you can inspect its configuration. This configuration sets up BGP sessions with other nodes, allowing for the distribution of routing information.

(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~  
/run/containerd/io.containerd.runtime.v2.task/k8s.io/034e5026927aae33059fe64deaf8a573bbb5491a605440ac1a5a4b915ffa68c6/rootfs/etc/calico/confd/config/bird.cfg  
/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/54/fs/etc/calico/confd/config/bird.cfg

(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~  
function apply\_communities ()  
{  
}

  
include "bird\_aggr.cfg";  
include "bird\_ipam.cfg";

router id 192.168.10.10;

  
protocol kernel {  
  learn;               
  persist;             
  scan time 2;         
  import all;  
  export filter calico\_kernel\_programming;   
  graceful restart;    
                       
                       
                       
                       
                       
  merge paths on;      
}

  
protocol device {  
  debug { states };  
  scan time 2;      
}

protocol direct {  
  debug { states };  
  interface -"cali\*", -"kube-ipvs\*", "\*";   
                                            
                                            
                                            
                                            
                                            
                                            
                                            
                                            
}

  
template bgp bgp\_template {  
  debug { states };  
  description "Connection to BGP peer";  
  local as 64512;  
  gateway recursive;   
  add paths on;  
  graceful restart;    
  connect delay time 2;  
  connect retry time 5;  
  error wait time 5,30;  
}

  
protocol bgp Mesh\_192\_168\_20\_100 from bgp\_template {  
  neighbor 192.168.20.100 as 64512;  
  source address 192.168.10.10;    
  import all;          
                       
  export filter {  
    calico\_export\_to\_bgp\_peers(true);  
    reject;  
  };    
  passive on;   
}

  
protocol bgp Mesh\_192\_168\_10\_101 from bgp\_template {  
  neighbor 192.168.10.101 as 64512;  
  source address 192.168.10.10;    
  import all;          
                       
  export filter {  
    calico\_export\_to\_bgp\_peers(true);  
    reject;  
  };    
  passive on;   
}

  
protocol bgp Mesh\_192\_168\_10\_102 from bgp\_template {  
  neighbor 192.168.10.102 as 64512;  
  source address 192.168.10.10;    
  import all;          
                       
  export filter {  
    calico\_export\_to\_bgp\_peers(true);  
    reject;  
  };    
  passive on;   
}

## Understanding Intra-Node Communication

When two Pods are scheduled on the same node, they can communicate directly without involving any tunnel interfaces. This direct communication is managed by IPTables `FORWARD` rules. The Calico CNI plugin creates `cali*` interfaces on the node, and with Proxy ARP settings, the Pods receive the MAC address of the gateway they should communicate with. The tunnel interface on the node is not involved in this intra-node communication.

![[Raw/Media/Resources/2ae2a6c92e03807387dcb6945c920c9a_MD5.png]]

To test intra-node communication, we’ll deploy two Pods on the same node by specifying the `nodeName` field in their configurations. Here are the YAML configurations for `pod1` and `pod2.`

apiVersion: v1  
kind: Pod  
metadata:  
  name: pod1  
spec:  
  nodeName: k8s-w1  

  containers:  
  \- name: pod1  
    image: nicolaka/netshoot    
    command: \["tail"\]  
    args: \["-f", "/dev/null"\]  
  terminationGracePeriodSeconds: 0  
\---  
apiVersion: v1  
kind: Pod  
metadata:  
  name: pod2  
spec:  
  nodeName: k8s-w1  
  containers:  
  \- name: pod2  
    image: nicolaka/netshoot  
    command: \["tail"\]  
    args: \["-f", "/dev/null"\]  
  terminationGracePeriodSeconds: 0

curl \-s \-O https:

(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# curl \-s \-O https:  
(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# kubectl apply \-f node1-pod2.yaml  
pod/pod1 created  
pod/pod2 created

![[Raw/Media/Resources/99b53ec8f77fb99885519fdf3c962f32_MD5.png]]

[https://namejsjeongkr.tistory.com/5](https://namejsjeongkr.tistory.com/5) [https://mokpolar.tistory.com/64](https://mokpolar.tistory.com/64)

![[Raw/Media/Resources/a65d02fa90347e1807b14728bd2a122b_MD5.png]]

After deploying the Pods, you may notice changes in the node’s network interfaces and routing tables:

-   New `cali*` Interfaces: Calico creates new network interfaces (e.g., `cali1234567890`) for each Pod.
-   Routing Table Entries: Two new routing table entries are added, routing traffic to the Pods’ IP addresses via the respective `cali*` interfaces.

root@k8s-w1:~# ip -c link  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
2: ens5: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 9001 qdisc mq state UP mode DEFAULT group default qlen 1000  
    link/ether 02:76:c7:5b:75:69 brd ff:ff:ff:ff:ff:ff  
    altname enp0s5  
3: tunl0@NONE: <NOARP,UP,LOWER\_UP> mtu 8981 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/ipip 0.0.0.0 brd 0.0.0.0  
6: cali1429ee5e74b@if3: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 8981 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-2a0b7452-fc2d-7a4d-be91-4e22eef29a0f  
7: cali1a6c68cfc2f@if3: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 8981 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-c36e8e9d-1ad0-4883\-2e09-50d1edd399ea  
8: cali7760be1255d@if3: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 8981 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-56441a7d-579a-877b-ed76-23226e7cd6a8  
9: calice0906292e2@if3: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 8981 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-a37fb783-52ac-b62e-8350\-ee7d7097cf73  
10: calibd2348b4f67@if3: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 8981 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-c744e9a9-de5d-377f-54b4-1c8ee5b50459

root@k8s-w1:~  
default via 192.168.10.1 dev ens5 proto dhcp src 192.168.10.101 metric 100  
172.16.34.0/24 via 192.168.20.100 dev tunl0 proto bird onlink  
172.16.116.0/24 via 192.168.10.10 dev tunl0 proto bird onlink  
blackhole 172.16.158.0/24 proto bird  
172.16.158.1 dev cali1429ee5e74b scope link  
172.16.158.2 dev cali1a6c68cfc2f scope link  
172.16.158.3 dev cali7760be1255d scope link  
172.16.158.4 dev calice0906292e2 scope link  
172.16.158.5 dev calibd2348b4f67 scope link  
172.16.184.0/24 via 192.168.10.102 dev tunl0 proto bird onlink  
192.168.0.2 via 192.168.10.1 dev ens5 proto dhcp src 192.168.10.101 metric 100  
192.168.10.0/24 dev ens5 proto kernel scope link src 192.168.10.101 metric 100  
192.168.10.1 dev ens5 proto dhcp scope link src 192.168.10.101 metric 100

The Pod has a default route via `169.254.1.1`, which is a virtual router provided by Calico for intra-node communication.

\# 네트워크 인터페이스 정보 확인 : calice#~ 2개 추가됨!  
\# 각각 net ns(네임스페이스) 0, 1로 호스트와 구별됨

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# kubectl exec \-it pod1 \-n default /bin/bash  
kubectl exec \[POD\] \[COMMAND\] is DEPRECATED and will be removed in a future version. Use kubectl exec \[POD\]   
pod1:~\# ip \-c link  
1: lo: <LOOPBACK,UP,LOWER\_UP\> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
2: tunl0@NONE: <NOARP\> mtu 1480 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ipip 0.0.0.0 brd 0.0.0.0  
3: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER\_UP\> mtu 8981 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether a2:06:d7:da:d4:18 brd ff:ff:ff:ff:ff:ff link\-netnsid 0

pod1:~\# ip \-c route  
default via 169.254.1.1 dev eth0  
169.254.1.1 dev eth0 scope link

pod1:~\# route \-n  
Kernel IP routing table  
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface  
0.0.0.0         169.254.1.1     0.0.0.0         UG    0      0        0 eth0  
169.254.1.1     0.0.0.0         255.255.255.255 UH    0      0        0 eth0

pod1:~  
PING 172.16.158.4 (172.16.158.4) 56(84) bytes of data.  
64 bytes from 172.16.158.4: icmp\_seq=1 ttl=64 time=0.019 ms  
64 bytes from 172.16.158.4: icmp\_seq=2 ttl=64 time=0.027 ms  
64 bytes from 172.16.158.4: icmp\_seq=3 ttl=64 time=0.028 ms  
64 bytes from 172.16.158.4: icmp\_seq=4 ttl=64 time=0.030 ms  
64 bytes from 172.16.158.4: icmp\_seq=5 ttl=64 time=0.033 ms  
64 bytes from 172.16.158.4: icmp\_seq=6 ttl=64 time=0.029 ms  
^C  
\--- 172.16.158.4 ping statistics ---  
6 packets transmitted, 6 received, 0% packet loss, time 5123ms  
rtt min/avg/max/mdev = 0.019/0.027/0.033/0.004 ms

## Testing Connectivity to the Other Pod

From `pod1`, ping `pod2` using its IP address (e.g., `172.16.158.4`): The successful ping confirms that Pods on the same node can communicate directly.

Understanding the Pod IP Address:

-   `172.16.158.4` is the IP address of `pod2`.
-   It belongs to the `172.16.158.0/24` subnet assigned by Calico.
-   The low latency indicates that the Pods are on the same node.

root@k8s-w1:~# tcpdump -i $VETH1 -nnip addr show  
tcpdump: \-nnip: No such device exists  
(SIOCGIFHWADDR: No such device)  
root@k8s-w1:~# ip addr show  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
    inet 127.0.0.1/8 scope host lo  
       valid\_lft forever preferred\_lft forever  
    inet6 ::1/128 scope host  
       valid\_lft forever preferred\_lft forever  
2: ens5: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 9001 qdisc mq state UP group default qlen 1000  
    link/ether 02:76:c7:5b:75:69 brd ff:ff:ff:ff:ff:ff  
    altname enp0s5  
    inet 192.168.10.101/24 metric 100 brd 192.168.10.255 scope global dynamic ens5  
       valid\_lft 1850sec preferred\_lft 1850sec  
    inet6 fe80::76:c7ff:fe5b:7569/64 scope link  
       valid\_lft forever preferred\_lft forever  
3: tunl0@NONE: <NOARP,UP,LOWER\_UP> mtu 8981 qdisc noqueue state UNKNOWN group default qlen 1000  
    link/ipip 0.0.0.0 brd 0.0.0.0  
    inet 172.16.158.0/32 scope global tunl0  
       valid\_lft forever preferred\_lft forever  
6: cali1429ee5e74b@if3: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 8981 qdisc noqueue state UP group default qlen 1000  
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-2a0b7452-fc2d-7a4d-be91-4e22eef29a0f  
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link  
       valid\_lft forever preferred\_lft forever  
7: cali1a6c68cfc2f@if3: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 8981 qdisc noqueue state UP group default qlen 1000  
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-c36e8e9d-1ad0-4883\-2e09-50d1edd399ea  
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link  
       valid\_lft forever preferred\_lft forever  
8: cali7760be1255d@if3: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 8981 qdisc noqueue state UP group default qlen 1000  
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-56441a7d-579a-877b-ed76-23226e7cd6a8  
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link  
       valid\_lft forever preferred\_lft forever  
9: calice0906292e2@if3: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 8981 qdisc noqueue state UP group default qlen 1000  
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-a37fb783-52ac-b62e-8350\-ee7d7097cf73  
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link  
       valid\_lft forever preferred\_lft forever  
10: calibd2348b4f67@if3: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 8981 qdisc noqueue state UP group default qlen 1000  
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-c744e9a9-de5d-377f-54b4-1c8ee5b50459  
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link  
       valid\_lft forever preferred\_lft forever  
root@k8s-w1:~#

To observe the network traffic between the Pods, we can use `tcpdump` on the host. On the node, list the network interfaces and identify the `cali*` interface corresponding to `pod1` or `pod2`. You can use `tcpdump` to capture network packets and observe the communication between the Pods.

root@k8s-w1:~# tcpdump \-i cali1429ee5e74b \-nn  
tcpdump: verbose output suppressed, use \-v\[v\]... for full protocol decode  
listening on cali1429ee5e74b, link-type EN10MB (Ethernet), snapshot length 262144 bytes  
02:16:36.785077 IP 192.168.10.101.32948 > 172.16.158.1.8080: Flags \[S\], seq 1995566909, win 62587, options \[mss 8941,sackOK,TS val 2096407014 ecr 0,nop,wscale 7\], length 0  
02:16:36.785102 IP 172.16.158.1.8080 > 192.168.10.101.32948: Flags \[S.\], seq 3291293760, ack 1995566910, win 62503, options \[mss 8941,sackOK,TS val 3214329542 ecr 2096407014,nop,wscale 7\], length 0  
02:16:36.785135 IP 192.168.10.101.32948 > 172.16.158.1.8080: Flags \[.\], ack 1, win 489, options \[nop,nop,TS val 2096407014 ecr 3214329542\], length 0  
02:16:36.785280 IP 192.168.10.101.42932 > 172.16.158.1.8181: Flags \[S\], seq 1567875025, win 62587, options \[mss 8941,sackOK,TS val 2096407014 ecr 0,nop,wscale 7\], length 0  
02:16:36.785289 IP 172.16.158.1.8181 > 192.168.10.101.42932: Flags \[S.\], seq 37504632, ack 1567875026, win 62503, options \[mss 8941,sackOK,TS val 3214329542 ecr 2096407014,nop,wscale 7\], length 0  
02:16:36.785303 IP 192.168.10.101.42932 > 172.16.158.1.8181: Flags \[.\], ack 1, win 489, options \[nop,nop,TS val 2096407014 ecr 3214329542\], length 0  
02:16:36.785417 IP 192.168.10.101.42932 > 172.16.158.1.8181: Flags \[P.\], seq 1:110, ack 1, win 489, options \[nop,nop,TS val 2096407014 ecr 3214329542\], length 109  
02:16:36.785424 IP 172.16.158.1.8181 > 192.168.10.101.42932: Flags \[.\], ack 110, win 488, options \[nop,nop,TS val 3214329542 ecr 2096407014\], length 0  
02:16:36.785455 IP 192.168.10.101.32948 > 172.16.158.1.8080: Flags \[P.\], seq 1:111, ack 1, win 489, options \[nop,nop,TS val 2096407014 ecr 3214329542\], length 110: HTTP: GET /health HTTP/1.1  
02:16:36.785459 IP 172.16.158.1.8080 > 192.168.10.101.32948: Flags \[.\], ack 111, win 488, options \[nop,nop,TS val 3214329542 ecr 2096407014\], length 0  
02:16:36.788880 IP 172.16.158.1.8181 > 192.168.10.101.42932: Flags \[P.\], seq 1:138, ack 110, win 488, options \[nop,nop,TS val 3214329546 ecr 2096407014\], length 137  
02:16:36.788934 IP 192.168.10.101.42932 > 172.16.158.1.8181: Flags \[.\], ack 138, win 488, options \[nop,nop,TS val 2096407018 ecr 3214329546\], length 0  
02:16:36.788976 IP 172.16.158.1.8181 > 192.168.10.101.42932: Flags \[F.\], seq 138, ack 110, win 488, options \[nop,nop,TS val 3214329546 ecr 2096407018\], length 0  
02:16:36.789087 IP 172.16.158.1.8080 > 192.168.10.101.32948: Flags \[P.\], seq 1:138, ack 111, win 488, options \[nop,nop,TS val 3214329546 ecr 2096407014\], length 137: HTTP: HTTP/1.1 200 OK  
02:16:36.789115 IP 192.168.10.101.32948 > 172.16.158.1.8080: Flags \[.\], ack 138, win 488, options \[nop,nop,TS val 2096407018 ecr 3214329546\], length 0  
02:16:36.789142 IP 172.16.158.1.8080 > 192.168.10.101.32948: Flags \[F.\], seq 138, ack 111, win 488, options \[nop,nop,TS val 3214329546 ecr 2096407018\], length 0  
02:16:36.789353 IP 192.168.10.101.32948 > 172.16.158.1.8080: Flags \[F.\], seq 111, ack 139, win 488, options \[nop,nop,TS val 2096407018 ecr 3214329546\], length 0  
02:16:36.789377 IP 172.16.158.1.8080 > 192.168.10.101.32948: Flags \[.\], ack 112, win 488, options \[nop,nop,TS val 3214329546 ecr 2096407018\], length 0  
02:16:36.789595 IP 192.168.10.101.42932 > 172.16.158.1.8181: Flags \[F.\], seq 110, ack 139, win 488, options \[nop,nop,TS val 2096407018 ecr 3214329546\], length 0  
02:16:36.789607 IP 172.16.158.1.8181 > 192.168.10.101.42932: Flags \[.\], ack 111, win 488, options \[nop,nop,TS val 3214329546 ecr 2096407018\], length 0

**Source and Destination IPs:**

-   `192.168.10.101`: Node's IP address.
-   `172.16.158.1`: Pod's IP address.

Ports:

-   Communication occurs over ports `8080` and `8181`.
-   The standard three-way handshake is observed.
-   An HTTP GET request is sent to `/health`, indicating a health check.
-   Connection Termination:
-   Proper connection termination with `FIN` packets.

## How Inter-Node Communication Works

![[Raw/Media/Resources/47542aecd69b422ce93e934d31cb03f9_MD5.png]]

[https://velog.io/@xgro/KANS3-3week#-step-02-calico-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EB%AA%A8%EB%93%9C](https://velog.io/@xgro/KANS3-3week#-step-02-calico-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EB%AA%A8%EB%93%9C)

When Pods are on different nodes, Calico uses IP-in-IP (IPIP) tunneling to facilitate communication. Here’s how it works:

-   BGP Advertisement: Each node advertises its Pod network range via BGP using the BIRD routing daemon.
-   Routing Table Updates: Calico’s Felix agent updates the host’s routing table with routes to the Pod networks on other nodes.
-   IPIP Tunnel: Traffic between Pods on different nodes is encapsulated in an IPIP tunnel. The outer IP header contains the source and destination node IPs, and the inner IP header contains the source and destination Pod IPs.

sudo tcpdump -i any proto 4 -w ipip\_traffic.pcap

## How Pods Access External Networks

When a Pod communicates with an external internet resource, the traffic is NAT’ed (Network Address Translation) using the node’s network interface IP. Here’s the process:

-   Outbound NAT: Calico’s default configuration enables outbound NAT (`natOutgoing: true` in the IP pool configuration).
-   IPTables MASQUERADE Rule: An IPTables `MASQUERADE` rule is applied, which replaces the source IP of the Pod with the node's IP when traffic exits the node.
-   No Tunnel Involvement: The tunnel interface is not involved in this communication.

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# iptables \-n \-t nat   
Chain cali\-nat\-outgoing (1 references)  
target     prot opt source               destination  
MASQUERADE  all  

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# curl \-O https://raw.githubusercontent.com/gasida/NDKS/main/4/node1\-pod1.yaml  
kubectl apply \-f node1\-pod1.yaml  
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current  
                                 Dload  Upload   Total   Spent    Left  Speed  
100   218  100   218    0     0    411      0   
pod/pod1 unchanged  
(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# kubectl get pods \-o wide  
NAME   READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES  
pod1   1/1     Running   0          16m   172.16.158.4   k8s\-w1   <none\>           <none\>  
pod2   1/1     Running   0          16m   172.16.158.5   k8s\-w1   <none\>           <none\>

Enter the Pod’s shell and attempt to reach an external service, such as Google’s DNS server. You will observe that the source IP address is the node’s IP, not the Pod’s IP. This confirms that NAT is translating the Pod’s IP to the node’s IP for outbound traffic.

root@k8s-w1:~  
Chain cali-nat-outgoing (1 references)  
 pkts bytes target     prot opt in     out     source               destination  
   11   756 MASQUERADE  all  --  \*      \*       0.0.0.0/0            0.0.0.0/0            /\* cali:flqWnvo8yq4ULQLa \*/ match\-set cali40masq-ipam-pools src ! match\-set cali40all-ipam-pools dst random-fully

pod1:~# ping \-c 10 8.8.8.8  
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.  
64 bytes from 8.8.8.8: icmp\_seq=1 ttl=104 time=33.5 ms  
64 bytes from 8.8.8.8: icmp\_seq=2 ttl=104 time=33.5 ms  
64 bytes from 8.8.8.8: icmp\_seq=3 ttl=104 time=33.5 ms  
64 bytes from 8.8.8.8: icmp\_seq=4 ttl=104 time=33.6 ms  
64 bytes from 8.8.8.8: icmp\_seq=5 ttl=104 time=33.5 ms  
64 bytes from 8.8.8.8: icmp\_seq=6 ttl=104 time=33.6 ms  
64 bytes from 8.8.8.8: icmp\_seq=7 ttl=104 time=33.5 ms  
64 bytes from 8.8.8.8: icmp\_seq=8 ttl=104 time=33.5 ms  
64 bytes from 8.8.8.8: icmp\_seq=9 ttl=104 time=33.5 ms  
64 bytes from 8.8.8.8: icmp\_seq=10 ttl=104 time=33.5 ms

\--- 8.8.8.8 ping statistics \---  
10 packets transmitted, 10 received, 0% packet loss, time 9014ms  
rtt min/avg/max/mdev \= 33.456/33.500/33.559/0.034 ms  
pod1:~# ping \-c 10 8.8.8.8  
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.  
64 bytes from 8.8.8.8: icmp\_seq=1 ttl=104 time=33.5 ms  
64 bytes from 8.8.8.8: icmp\_seq=2 ttl=104 time=33.5 ms  
64 bytes from 8.8.8.8: icmp\_seq=3 ttl=104 time=33.5 ms  
64 bytes from 8.8.8.8: icmp\_seq=4 ttl=104 time=33.5 ms  
64 bytes from 8.8.8.8: icmp\_seq=5 ttl=104 time=33.5 ms  
64 bytes from 8.8.8.8: icmp\_seq=6 ttl=104 time=33.5 ms  
64 bytes from 8.8.8.8: icmp\_seq=7 ttl=104 time=33.5 ms  
64 bytes from 8.8.8.8: icmp\_seq=8 ttl=104 time=33.5 ms  
64 bytes from 8.8.8.8: icmp\_seq=9 ttl=104 time=33.5 ms  
64 bytes from 8.8.8.8: icmp\_seq=10 ttl=104 time=33.5 ms

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# kubectl get pods \-o wide  
NAME   READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES  
pod1   1/1     Running   0          18m   172.16.158.4   k8s\-w1   <none\>           <none\>  
pod2   1/1     Running   0          18m   172.16.158.5   k8s\-w1   <none\>           <none\>  
(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# kubectl get nodes \-o wide  
NAME     STATUS   ROLES           AGE   VERSION   INTERNAL\-IP      EXTERNAL\-IP   OS\-IMAGE             KERNEL\-VERSION   CONTAINER\-RUNTIME  
k8s\-m    Ready    control\-plane   34m   v1.30.5   192.168.10.10    <none\>        Ubuntu 22.04.5 LTS   6.5.0\-1024\-aws   containerd://1.7.22  
k8s\-w0   Ready    <none\>          33m   v1.30.5   192.168.20.100   <none\>        Ubuntu 22.04.5 LTS   6.5.0\-1024\-aws   containerd://1.7.22  
k8s\-w1   Ready    <none\>          33m   v1.30.5   192.168.10.101   <none\>        Ubuntu 22.04.5 LTS   6.5.0\-1024\-aws   containerd://1.7.22  
k8s\-w2   Ready    <none\>          33m   v1.30.5   192.168.10.102   <none\>        Ubuntu 22.04.5 LTS   6.5.0\-1024\-aws   containerd://1.7.22

02:20:46.788390 IP 172.16.158.1.8181 \> 192.168.10.101.58390: Flags \[F.\], seq 138, ack 110, win 488, options \[nop,nop,TS val 3214579545 ecr 2096657017\], length 0  
02:20:46.788390 IP 172.16.158.1.8080 \> 192.168.10.101.49504: Flags \[F.\], seq 138, ack 111, win 488, options \[nop,nop,TS val 3214579545 ecr 2096657017\], length 0  
02:20:46.788529 IP 192.168.10.101.58390 \> 172.16.158.1.8181: Flags \[F.\], seq 110, ack 139, win 488, options \[nop,nop,TS val 2096657017 ecr 3214579545\], length 0  
02:20:46.788545 IP 172.16.158.1.8181 \> 192.168.10.101.58390: Flags \[.\], ack 111, win 488, options \[nop,nop,TS val 3214579545 ecr 2096657017\], length 0  
02:20:46.788639 IP 192.168.10.101.49504 \> 172.16.158.1.8080: Flags \[F.\], seq 111, ack 139, win 488, options \[nop,nop,TS val 2096657017 ecr 3214579545\], length 0  
02:20:46.788649 IP 172.16.158.1.8080 \> 192.168.10.101.49504: Flags \[.\], ack 112, win 488, options \[nop,nop,TS val 3214579545 ecr 2096657017\], length 0  
02:20:51.965220 IP 10.200.1.1.443 \> 172.16.158.1.55696: Flags \[P.\], seq 385:503, ack 157, win 472, options \[nop,nop,TS val 760412201 ecr 2186959604\], length 118  
02:20:51.965241 IP 172.16.158.1.55696 \> 10.200.1.1.443: Flags \[.\], ack 503, win 443, options \[nop,nop,TS val 2186969545 ecr 760412201\], length 0  
02:20:56.784864 IP 192.168.10.101.60740 \> 172.16.158.1.8080: Flags \[S\], seq 1174607584, win 62587, options \[mss 8941,sackOK,TS val 2096667014 ecr 0,nop,wscale 7\], length 0  
02:20:56.784886 IP 172.16.158.1.8080 \> 192.168.10.101.60740: Flags \[S.\], seq 775339285, ack 1174607585, win 62503, options \[mss 8941,sackOK,TS val 3214589542 ecr 2096667014,nop,wscale 7\], length 0  
02:20:56.784912 IP 192.168.10.101.60740 \> 172.16.158.1.8080: Flags \[.\], ack 1, win 489, options \[nop,nop,TS val 2096667014 ecr 3214589542\], length 0  
02:20:56.785048 IP 192.168.10.101.51100 \> 172.16.158.1.8181: Flags \[S\], seq 1218467495, win 62587, options \[mss 8941,sackOK,TS val 2096667014 ecr 0,nop,wscale 7\], length 0  
02:20:56.785056 IP 172.16.158.1.8181 \> 192.168.10.101.51100: Flags \[S.\], seq 1228248012, ack 1218467496, win 62503, options \[mss 8941,sackOK,TS val 3214589542 ecr 2096667014,nop,wscale 7\], length 0  
02:20:56.785069 IP 192.168.10.101.51100 \> 172.16.158.1.8181: Flags \[.\], ack 1, win 489, options \[nop,nop,TS val 2096667014 ecr 3214589542\], length 0  
02:20:56.785291 IP 192.168.10.101.51100 \> 172.16.158.1.8181: Flags \[P.\], seq 1:110, ack 1, win 489, options \[nop,nop,TS val 2096667014 ecr 3214589542\], length 109  
02:20:56.785300 IP 172.16.158.1.8181 \> 192.168.10.101.51100: Flags \[.\], ack 110, win 488, options \[nop,nop,TS val 3214589542 ecr 2096667014\], length 0  
02:20:56.785352 IP 192.168.10.101.60740 \> 172.16.158.1.8080: Flags \[P.\], seq 1:111, ack 1, win 489, options \[nop,nop,TS val 2096667014 ecr 3214589542\], length 110: HTTP: GET /health HTTP/1.1  
02:20:56.785358 IP 172.16.158.1.8080 \> 192.168.10.101.60740: Flags \[.\], ack 111, win 488, options \[nop,nop,TS val 3214589542 ecr 2096667014\], length 0  
02:20:56.789029 IP 172.16.158.1.8080 \> 192.168.10.101.60740: Flags \[P.\], seq 1:138, ack 111, win 488, options \[nop,nop,TS val 3214589546 ecr 2096667014\], length 137: HTTP: HTTP/1.1 200 OK  
02:20:56.789029 IP 172.16.158.1.8181 \> 192.168.10.101.51100: Flags \[P.\], seq 1:138, ack 110, win 488, options \[nop,nop,TS val 3214589546 ecr 2096667014\], length 137  
02:20:56.789061 IP 192.168.10.101.60740 \> 172.16.158.1.8080: Flags \[.\], ack 138, win 488, options \[nop,nop,TS val 2096667018 ecr 3214589546\], length 0  
02:20:56.789061 IP 192.168.10.101.51100 \> 172.16.158.1.8181: Flags \[.\], ack 138, win 488, options \[nop,nop,TS val 2096667018 ecr 3214589546\], length 0  
02:20:56.789089 IP 172.16.158.1.8181 \> 192.168.10.101.51100: Flags \[F.\], seq 138, ack 110, win 488, options \[nop,nop,TS val 3214589546 ecr 2096667018\], length 0  
02:20:56.789089 IP 172.16.158.1.8080 \> 192.168.10.101.60740: Flags \[F.\], seq 138, ack 111, win 488, options \[nop,nop,TS val 3214589546 ecr 2096667018\], length 0  
02:20:56.789224 IP 192.168.10.101.60740 \> 172.16.158.1.8080: Flags \[F.\], seq 111, ack 139, win 488, options \[nop,nop,TS val 2096667018 ecr 3214589546\], length 0  
02:20:56.789239 IP 172.16.158.1.8080 \> 192.168.10.101.60740: Flags \[.\], ack 112, win 488, options \[nop,nop,TS val 3214589546 ecr 2096667018\], length 0  
02:20:56.789360 IP 192.168.10.101.51100 \> 172.16.158.1.8181: Flags \[F.\], seq 110, ack 139, win 488, options \[nop,nop,TS val 2096667018 ecr 3214589546\], length 0  
02:20:56.789370 IP 172.16.158.1.8181 \> 192.168.10.101.51100: Flags \[.\], ack 111, win 488, options \[nop,nop,TS val 3214589546 ecr 2096667018\], length 0  
02:21:06.784912 IP 192.168.10.101.48680 \> 172.16.158.1.8080: Flags \[S\], seq 1952863696, win 62587, options \[mss 8941,sackOK,TS val 2096677014 ecr 0,nop,wscale 7\], length 0  
02:21:06.784936 IP 172.16.158.1.8080 \> 192.168.10.101.48680: Flags \[S.\], seq 3873586601, ack 1952863697, win 62503, options \[mss 8941,sackOK,TS val 3214599542 ecr 2096677014,nop,wscale 7\], length 0  
02:21:06.784965 IP 192.168.10.101.48680 \> 172.16.158.1.8080: Flags \[.\], ack 1, win 489, options \[nop,nop,TS val 2096677014 ecr 3214599542\], length 0  
02:21:06.785407 IP 192.168.10.101.48680 \> 172.16.158.1.8080: Flags \[P.\], seq 1:111, ack 1, win 489, options \[nop,nop,TS val 2096677014 ecr 3214599542\], length 110: HTTP: GET /health HTTP/1.1  
02:21:06.785422 IP 172.16.158.1.8080 \> 192.168.10.101.48680: Flags \[.\], ack 111, win 488, options \[nop,nop,TS val 3214599542 ecr 2096677014\], length 0  
02:21:06.785542 IP 172.16.158.1.8080 \> 192.168.10.101.48680: Flags \[P.\], seq 1:138, ack 111, win 488, options \[nop,nop,TS val 3214599542 ecr 2096677014\], length 137: HTTP: HTTP/1.1 200 OK  
02:21:06.785574 IP 172.16.158.1.8080 \> 192.168.10.101.48680: Flags \[F.\], seq 138, ack 111, win 488, options \[nop,nop,TS val 3214599542 ecr 2096677014\], length 0  
02:21:06.785627 IP 192.168.10.101.36582 \> 172.16.158.1.8181: Flags \[S\], seq 1725663695, win 62587, options \[mss 8941,sackOK,TS val 2096677014 ecr 0,nop,wscale 7\], length 0  
02:21:06.785636 IP 172.16.158.1.8181 \> 192.168.10.101.36582: Flags \[S.\], seq 2364610350, ack 1725663696, win 62503, options \[mss 8941,sackOK,TS val 3214599542 ecr 2096677014,nop,wscale 7\], length 0  
02:21:06.785649 IP 192.168.10.101.36582 \> 172.16.158.1.8181: Flags \[.\], ack 1, win 489, options \[nop,nop,TS val 2096677014 ecr 3214599542\], length 0  
02:21:06.785735 IP 192.168.10.101.36582 \> 172.16.158.1.8181: Flags \[P.\], seq 1:110, ack 1, win 489, options \[nop,nop,TS val 2096677014 ecr 3214599542\], length 109  
02:21:06.785740 IP 172.16.158.1.8181 \> 192.168.10.101.36582: Flags \[.\], ack 110, win 488, options \[nop,nop,TS val 3214599542 ecr 2096677014\], length 0  
02:21:06.785886 IP 172.16.158.1.8181 \> 192.168.10.101.36582: Flags \[P.\], seq 1:138, ack 110, win 488, options \[nop,nop,TS val 3214599543 ecr 2096677014\], length 137  
02:21:06.785906 IP 192.168.10.101.36582 \> 172.16.158.1.8181: Flags \[.\], ack 138, win 488, options \[nop,nop,TS val 2096677015 ecr 3214599543\], length 0  
02:21:06.785946 IP 172.16.158.1.8181 \> 192.168.10.101.36582: Flags \[F.\], seq 138, ack 110, win 488, options \[nop,nop,TS val 3214599543 ecr 2096677015\], length 0  
02:21:06.785975 IP 192.168.10.101.48680 \> 172.16.158.1.8080: Flags \[.\], ack 138, win 488, options \[nop,nop,TS val 2096677015 ecr 3214599542\], length 0  
02:21:06.786134 IP 192.168.10.101.48680 \> 172.16.158.1.8080: Flags \[F.\], seq 111, ack 139, win 488, options \[nop,nop,TS val 2096677015 ecr 3214599542\], length 0  
02:21:06.786147 IP 172.16.158.1.8080 \> 192.168.10.101.48680: Flags \[.\], ack 112, win 488, options \[nop,nop,TS val 3214599543 ecr 2096677015\], length 0  
02:21:06.786246 IP 192.168.10.101.36582 \> 172.16.158.1.8181: Flags \[F.\], seq 110, ack 139, win 488, options \[nop,nop,TS val 2096677015 ecr 3214599543\], length 0  
02:21:06.786255 IP 172.16.158.1.8181 \> 192.168.10.101.36582: Flags \[.\], ack 111, win 488, options \[nop,nop,TS val 3214599543 ecr 2096677015\], length 0

Calico offers several networking modes to accommodate various environments and requirements:

## a. IP-in-IP (IPIP) Mode

-   Description: Uses IP-in-IP encapsulation to tunnel Pod traffic between nodes.
-   Usage: Default mode in Calico; suitable when direct routing between nodes is not possible.
-   Interface: Utilizes the `tunl0` interface for encapsulated packets.
-   Pros: Simplifies network configuration; no need for underlying network changes.
-   Cons: Adds overhead due to encapsulation; not supported in some cloud environments like Azure.

## b. Direct Routing Mode

-   Description: Pods communicate directly without encapsulation, using the underlying network for routing.
-   Usage: Preferred for performance when nodes are on the same Layer 2 network and can route Pod CIDRs.
-   Pros: Best performance; reduced overhead.
-   Cons: Requires network infrastructure to route Pod CIDRs; may not be feasible in all environments.

## c. CrossSubnet Mode

-   Description: Combines IPIP and Direct modes. Uses direct routing within the same subnet and IPIP tunneling across different subnets.
-   Usage: Ideal for environments with multiple subnets where intra-subnet communication can be direct.
-   Pros: Balances performance and compatibility.
-   Cons: Requires proper subnet configuration and understanding of network topology.

## d. VXLAN Mode

-   Description: Uses VXLAN encapsulation to tunnel traffic between nodes, encapsulating Layer 2 frames over UDP.
-   Usage: Useful in environments where IPIP is not supported (e.g., Azure).
-   Pros: Supported in more environments; handles broadcast and multicast traffic.
-   Cons: Slightly more complex; adds UDP overhead.

## e. WireGuard Encryption

-   Description: Encrypts Pod-to-Pod traffic between nodes using WireGuard VPN.
-   Usage: Enhances security by encrypting traffic over any of the above modes.
-   Pros: Adds encryption without significant performance impact.
-   Cons: Requires additional configuration; not all environments support WireGuard natively.

In AWS, by default, the network interface card (NIC) drops any traffic that doesn’t originate from or is not destined to the instance’s IP address. Since Calico assigns IP addresses to pods that are different from the node’s IP, we need to disable the Source/Destination Check feature on the EC2 instances to allow traffic to and from pod IPs.

aws ec2 modify-instance-attribute \--instance-id <INSTANCE\_ID> \--source-dest-check "{\\"Value\\": false}"

![[Raw/Media/Resources/cdb4b6b725e2c338f3f291b4b7b15add_MD5.png]]

By default, Calico uses IP-in-IP (IPIP) encapsulation for cross-node pod communication. In Direct mode, Calico disables encapsulation, and traffic between nodes is routed directly, assuming the underlying network supports routing pod IPs.

First, check the current IPPool configuration: Here, `IPIPMODE` is set to `Always`, meaning IPIP encapsulation is always used. To switch to Direct mode, we need to change `IPIPMODE` to `Never.` On the node, check the routing table to see if the routes now use the physical interface (e.g., `ens5` or `enp0s8`) instead of the tunnel interface (`tunl0`).

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# calicoctl get ippool \-o wide  
NAME                  CIDR            NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR  
default\-ipv4\-ippool   172.16.0.0/16   true   Always     Never       false      false              all()

Every 2.0s: route \-n | egrep '(Destination|UG)'                    k8s\-w1: Sun Sep 22 02:33:00 2024

Destination     Gateway         Genmask         Flags Metric Ref    Use Iface  
0.0.0.0         192.168.10.1    0.0.0.0         UG    100    0        0 ens5  
172.16.34.0     192.168.20.100  255.255.255.0   UG    0      0        0 tunl0  
172.16.116.0    192.168.10.10   255.255.255.0   UG    0      0        0 tunl0  
172.16.184.0    192.168.10.102  255.255.255.0   UG    0      0        0 tunl0  
192.168.0.2     192.168.10.1    255.255.255.255 UGH   100    0        0 ens5

(⎈|kubernetes\-admin@kubernetes:default) root@k8s\-m:~\# calicoctl get ippool default\-ipv4\-ippool \-o yaml | sed \-e "s/ipipMode: Always/ipipMode: Never/" | calicoctl apply \-f \-  
Successfully applied 1 'IPPool' resource(s)

To switch to Direct mode, we need to change IPIPMODE to Never:

(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# calicoctl get ippool -o wide  
NAME                  CIDR            NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR  
default\-ipv4-ippool   172.16.0.0/16   true   Never      Never       false      false              all()

route -n | egrep '(Destination|UG)'

ubuntu@k8s-w1:~$ route -n | egrep '(Destination|UG)'  
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface  
0.0.0.0         192.168.10.1    0.0.0.0         UG    100    0        0 ens5  
172.16.34.0     192.168.10.1    255.255.255.0   UG    0      0        0 ens5  
172.16.116.0    192.168.10.10   255.255.255.0   UG    0      0        0 ens5  
172.16.184.0    192.168.10.102  255.255.255.0   UG    0      0        0 ens5  
192.168.0.2     192.168.10.1    255.255.255.255 UGH   100    0        0 ens5

Create a new pod to test communication, List the pods and their IP addresses. Use `calicoctl` to check the workload endpoints:

\# 파드 생성  
curl \-s \-O https:  
kubectl apply \-f node3-pod3.yaml

\# 파드 IP 정보 확인  
kubectl get pod \-o wide  
calicoctl get wep  
(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# calicoctl get wep  
WORKLOAD   NODE     NETWORKS          INTERFACE  
pod1       k8s-w1   172.16.158.6/32   calice0906292e2  
pod2       k8s-w1   172.16.158.5/32   calibd2348b4f67  
pod3       k8s-w0   172.16.34.1/32    cali49778cadcf1

kubectl exec \-it pod1 \-- zsh

  
  
 pod1  ~  ping 172.16.158.6  
PING 172.16.158.6 (172.16.158.6) 56(84) bytes of data.  
64 bytes from 172.16.158.6: icmp\_seq=1 ttl=64 time=0.017 ms  
64 bytes from 172.16.158.6: icmp\_seq=2 ttl=64 time=0.029 ms  
64 bytes from 172.16.158.6: icmp\_seq=3 ttl=64 time=0.027 ms  
64 bytes from 172.16.158.6: icmp\_seq=4 ttl=64 time=0.028 ms  
64 bytes from 172.16.158.6: icmp\_seq=5 ttl=64 time=0.030 ms  
64 bytes from 172.16.158.6: icmp\_seq=6 ttl=64 time=0.026 ms  
64 bytes from 172.16.158.6: icmp\_seq=7 ttl=64 time=0.029 ms  
64 bytes from 172.16.158.6: icmp\_seq=8 ttl=64 time=0.029 ms  
64 bytes from 172.16.158.6: icmp\_seq=9 ttl=64 time=0.037 ms  
64 bytes from 172.16.158.6: icmp\_seq=10 ttl=64 time=0.076 ms  
64 bytes from 172.16.158.6: icmp\_seq=11 ttl=64 time=0.063 ms  
64 bytes from 172.16.158.6: icmp\_seq=12 ttl=64 time=0.027 ms  
64 bytes from 172.16.158.6: icmp\_seq=13 ttl=64 time=0.063 ms  
64 bytes from 172.16.158.6: icmp\_seq=14 ttl=64 time=0.028 ms  
64 bytes from 172.16.158.6: icmp\_seq=15 ttl=64 time=0.035 ms  
64 bytes from 172.16.158.6: icmp\_seq=16 ttl=64 time=0.031 ms  
64 bytes from 172.16.158.6: icmp\_seq=17 ttl=64 time=0.030 ms  
64 bytes from 172.16.158.6: icmp\_seq=18 ttl=64 time=0.031 ms  
64 bytes from 172.16.158.6: icmp\_seq=19 ttl=64 time=0.030 ms

**Pods Involved:**

-   `pod1` with IP 172.16.158.6
-   `pod2` with IP 172.16.158.5

**Node:**

-   Both Pods are running on the same node, `k8s-w1`.
-   An attempt to capture ICMP packets on the node’s physical network interface `ens5` using `tcpdump` resulted in no packets being captured.
-   Direct Communication: Since `pod1` and `pod2` are on the same node, their network traffic does not need to leave the node. The communication occurs internally, routed directly between the Pods without involving the node's physical network interface (`ens5`).
-   VEach Pod is connected to the node’s network stack via virtual Ethernet interfaces (veth pairs). The host side of these pairs are the `cali*` interfaces created by Calico (e.g., `calice0906292e2`, `calibd2348b4f67`).
-   The Linux kernel handles packet forwarding between these virtual interfaces internally. The traffic stays within the node’s network namespace, avoiding the overhead of external routing.

On the node `k8s-w1`, try capturing ICMP packets on the physical interface. Since traffic between Pods on the same node doesn’t pass through the physical interface, you need alternative methods to capture and analyze it.

1\.  pod1  ~  ping 172.16.158.5  
2\. (⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~  
NAME   READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES  
pod1   1/1     Running   0          16m   172.16.158.6   k8s-w1   <none>           <none>  
pod2   1/1     Running   0          47m   172.16.158.5   k8s-w1   <none>           <none>  
pod3   1/1     Running   0          16m   172.16.34.1    k8s-w0   <none>           <none>  
3\. root@k8s-w1:~  
tcpdump: listening on ens5, link-type EN10MB (Ethernet), snapshot length 262144 bytes  
^C0 packets captured  
0 packets received by filter  
0 packets dropped by kernel  
root@k8s-w1:~  
tcpdump: data link type LINUX\_SLL2  
tcpdump: listening on any, link-type LINUX\_SLL2 (Linux cooked v2), snapshot length 262144 bytes  
^C12 packets captured  
17 packets received by filter  
0 packets dropped by kernel  
root@k8s-w1:~  
tcpdump: data link type LINUX\_SLL2  
tcpdump: listening on any, link-type LINUX\_SLL2 (Linux cooked v2), snapshot length 262144 bytes  
^C168 packets captured  
173 packets received by filter  
0 packets dropped by kernel  
root@k8s-w1:~

root@k8s-w1:~  
reading from file /root/calico-all.pcap, link-type LINUX\_SLL2 (Linux cooked v2), snapshot length 262144  
Warning: interface names might be incorrect  
02:56:34.426876 calice0906292e2 In  IP 172.16.158.6 > 172.16.158.5: ICMP echo request, id 102, seq 7, length 64  
02:56:34.426910 calibd2348b4f67 Out IP 172.16.158.6 > 172.16.158.5: ICMP echo request, id 102, seq 7, length 64  
02:56:34.426919 calibd2348b4f67 In  IP 172.16.158.5 > 172.16.158.6: ICMP echo reply, id 102, seq 7, length 64  
02:56:34.426926 calice0906292e2 Out IP 172.16.158.5 > 172.16.158.6: ICMP echo reply, id 102, seq 7, length 64  
02:56:35.450857 calice0906292e2 In  IP 172.16.158.6 > 172.16.158.5: ICMP echo request, id 102, seq 8, length 64  
02:56:35.450890 calibd2348b4f67 Out IP 172.16.158.6 > 172.16.158.5: ICMP echo request, id 102, seq 8, length 64  
02:56:35.450900 calibd2348b4f67 In  IP 172.16.158.5 > 172.16.158.6: ICMP echo reply, id 102, seq 8, length 64  
02:56:35.450907 calice0906292e2 Out IP 172.16.158.5 > 172.16.158.6: ICMP echo reply, id 102, seq 8, length 64  
02:56:36.474884 calice0906292e2 In  IP 172.16.158.6 > 172.16.158.5: ICMP echo request, id 102, seq 9, length 64

We have `pod1` running on `k8s-w1` with IP `172.16.158.6` and `pod3` running on `k8s-w0` with IP `172.16.34.1`. Test communication between Pods on different nodes to understand inter-node networking behavior is our objective in this stage.

kubectl exec -it pod1 - zsh

pod1  ~  ping 172.16.34.1  
PING 172.16.34.1 (172.16.34.1) 56(84) bytes of data.  
^C  
\--- 172.16.34.1 ping statistics ---  
38 packets transmitted, 0 received, 100% packet loss, time 37890ms

ssh root@k8s-w1  
tcpdump -i ens5 -nn icmp

03:00:43.770862 IP 172.16.158.6 > 172.16.34.1: ICMP echo request, id 116, seq 26, length 64  
03:00:44.807303 IP 172.16.158.6 > 172.16.34.1: ICMP echo request, id 116, seq 27, length 64  
03:00:45.818854 IP 172.16.158.6 > 172.16.34.1: ICMP echo request, id 116, seq 28, length 64  
03:00:46.842842 IP 172.16.158.6 > 172.16.34.1: ICMP echo request, id 116, seq 29, length 64  
03:00:47.866979 IP 172.16.158.6 > 172.16.34.1: ICMP echo request, id 116, seq 30, length 64  
03:00:48.890853 IP 172.16.158.6 > 172.16.34.1: ICMP echo request, id 116, seq 31, length 64  
03:00:49.914933 IP 172.16.158.6 > 172.16.34.1: ICMP echo request, id 116, seq 32, length 64  
03:00:50.938932 IP 172.16.158.6 > 172.16.34.1: ICMP echo request, id 116, seq 33, length 64  
03:00:51.966849 IP 172.16.158.6 > 172.16.34.1: ICMP echo request, id 116, seq 34, length 64  
03:00:52.986853 IP 172.16.158.6 > 172.16.34.1: ICMP echo request, id 116, seq 35, length 64

When attempting to ping `pod3` from `pod1`, the ping requests time out. Capturing ICMP packets on `k8s-w1` shows that ping requests are sent but no replies are received. The worker nodes `k8s-w1` and `k8s-w0` are on different subnets, which can affect inter-node communication.

![[Raw/Media/Resources/08994da272bcba783157a98311b98ba7_MD5.png]]

**Using CrossSubnet Mode**: Enable direct routing within the same subnet and use IPIP encapsulation for traffic across different subnets. Routes within the same subnet use the main interface, while routes to different subnets use `tunl0`.

  
calicoctl patch ippool default-ipv4-ippool -p '{"spec":{"ipipMode":"CrossSubnet"}}'

\# 모드 확인  
calicoctl get ippool -o wide  
NAME                  CIDR            NAT    IPIPMODE      VXLANMODE   DISABLED   SELECTOR  
default-ipv4-ippool   172.16.0.0/16   true   CrossSubnet   Never       false      all()\# 파드 생성  
kubectl apply -f node3-pod3.yaml  
calicoctl get wep\# 호스트 라우팅 정보 확인.   
route -n | grep UG  
root@ip-172-20-63-146:~# route -n | grep UG  
100.105.79.128  172.20.61.184   255.255.255.192 UG    0      0        0 ens5  # 노드간 같은 네트워크 대역 - Direct 모드  
100.125.78.64   172.20.59.153   255.255.255.192 UG    0      0        0 ens5  # 노드간 같은 네트워크 대역 - Direct 모드  
100.127.64.128  172.20.64.181   255.255.255.192 UG    0      0        0 tunl0 # 노드간 다른 네트워크 대역 - IPIP 모드\# 파드 Shell 접속(zsh)  
kubectl exec -it pod1 -- zsh  
\## 파드 Shell 에서 아래 입력  
ping <pod2 혹은 pod3 IP>

-   Ping Pods on Same Subnet: Should work without encapsulation.
-   Ping Pods on Different Subnets: Should work with IPIP encapsulation.

![[Raw/Media/Resources/443e96367aa64d55c2af02f9c69568c9_MD5.png]]

Enabling WireGuard Encryption: Capture traffic on the main interface to confirm encryption **(packets should not reveal Pod IPs)**.

(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# calicoctl patch felixconfiguration default --type='merge' -p '{"spec":{"wireguardEnabled":true}}'  
Successfully patched 1 'FelixConfiguration' resource  
(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# calicoctl get felixconfiguration default -o yaml | grep wireguardEnabled  
  wireguardEnabled: true  
(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# calicoctl get felixconfiguration default -o yaml | grep wireguardEnabled  
  wireguardEnabled: true  
(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# calicoctl get node -o yaml | grep wireguardPublicKey  
    wireguardPublicKey: nGJDPnehiHNMwf9Z25Q8hGN1PCTeCTefh1pNAEjiPC8=  
    wireguardPublicKey: DwdcNTZ9CzJdaW5SxmvfrHNqDcHZgFTUqhf+srC/1yk=  
    wireguardPublicKey: rWepcpPRToZHtb563t5AKAhIHT69rMgTOIuCE8o3AgE=  
    wireguardPublicKey: XK6OS/pP/4/ntGg3WdPYTQP9SQLWNM884DlC/8OgK1s=  
(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# calicoctl get node k8s-w1 -o yaml | grep wireguardPublicKey  
  wireguardPublicKey: rWepcpPRToZHtb563t5AKAhIHT69rMgTOIuCE8o3AgE=  
(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# ip -c -d addr show wireguard.cali  
12: wireguard.cali: <POINTOPOINT,NOARP,UP,LOWER\_UP> mtu 8941 qdisc noqueue state UNKNOWN group default qlen 1000  
    link/none  promiscuity 0 minmtu 0 maxmtu 2147483552  
    wireguard numtxqueues 1 numrxqueues 1 gso\_max\_size 65536 gso\_max\_segs 65535  
    inet 172.16.116.2/32 scope global wireguard.cali  
       valid\_lft forever preferred\_lft forever  
(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# ifconfig wireguard.cali  
wireguard.cali: flags=209<UP,POINTOPOINT,RUNNING,NOARP>  mtu 8941  
        inet 172.16.116.2  netmask 255.255.255.255  destination 172.16.116.2  
        unspec 00\-00\-00\-00\-00\-00\-00\-00\-00\-00\-00\-00\-00\-00\-00\-00  txqueuelen 1000  (UNSPEC)  
        RX packets 0  bytes 0 (0.0 B)  
        RX errors 0  dropped 0  overruns 0  frame 0  
        TX packets 0  bytes 0 (0.0 B)  
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# wg showconf wireguard.cali  
\[Interface\]  
ListenPort = 51820  
FwMark = 0x100000  
PrivateKey = yF8F44A8v5Spr9jX5YwRSCKktNkPvkoYbLd1nuG8i0M=

\[Peer\]  
PublicKey = rWepcpPRToZHtb563t5AKAhIHT69rMgTOIuCE8o3AgE=  
AllowedIPs = 172.16.158.0/24, 172.16.158.7/32, 172.16.158.9/32  
Endpoint = 192.168.10.101:51820

\[Peer\]  
PublicKey = DwdcNTZ9CzJdaW5SxmvfrHNqDcHZgFTUqhf+srC/1yk=  
AllowedIPs = 172.16.34.2/32, 172.16.34.0/24, 172.16.34.4/32  
Endpoint = 192.168.20.100:51820

\[Peer\]  
PublicKey = XK6OS/pP/4/ntGg3WdPYTQP9SQLWNM884DlC/8OgK1s=  
AllowedIPs = 172.16.184.0/24, 172.16.184.1/32, 172.16.184.3/32  
Endpoint = 192.168.10.102:51820  
(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~# wg show  
interface: wireguard.cali  
  public key: nGJDPnehiHNMwf9Z25Q8hGN1PCTeCTefh1pNAEjiPC8=  
  private key: (hidden)  
  listening port: 51820  
  fwmark: 0x100000

peer: rWepcpPRToZHtb563t5AKAhIHT69rMgTOIuCE8o3AgE=  
  endpoint: 192.168.10.101:51820  
  allowed ips: 172.16.158.0/24, 172.16.158.7/32, 172.16.158.9/32

peer: DwdcNTZ9CzJdaW5SxmvfrHNqDcHZgFTUqhf+srC/1yk=  
  endpoint: 192.168.20.100:51820  
  allowed ips: 172.16.34.2/32, 172.16.34.0/24, 172.16.34.4/32

peer: XK6OS/pP/4/ntGg3WdPYTQP9SQLWNM884DlC/8OgK1s=  
  endpoint: 192.168.10.102:51820  
  allowed ips: 172.16.184.0/24, 172.16.184.1/32, 172.16.184.3/32  
(⎈|kubernetes-admin@kubernetes:default) root@k8s-m:~#

To maintain a healthy Kubernetes cluster with Calico networking, it’s crucial to monitor the Calico components for performance and issues. You can use Prometheus to collect valuable metrics from Calico components such as Felix and Typha, as well as kube-controllers.

![[Raw/Media/Resources/3947e235cadc21781c66b4d74914089a_MD5.png]]

## Enable Metrics in Felix

First, enable Prometheus metrics in the Felix configuration: By default, Felix publishes its metrics on TCP port 9091. Create a Kubernetes Service to expose these metrics.

Configure Calico to enable metrics reporting

  
calicoctl get felixconfiguration \-o yaml  
calicoctl patch felixconfiguration default  \--patch '{"spec":{"prometheusMetricsEnabled": true}}'

  
kubectl apply \-f \- <<EOF  
apiVersion: v1  
kind: Service  
metadata:  
  name: felix-metrics-svc  
  namespace: kube-system  
spec:  
  clusterIP: None  
  selector:  
    k8s-app: calico-node  
  ports:  
  \- port: 9091  
    targetPort: 9091  
EOF  
kubectl get svc,ep \-n kube-system felix-metrics-svc

  
  
calicoctl get kubecontrollersconfiguration default \-o yaml  
kubectl apply \-f \- <<EOF  
apiVersion: v1  
kind: Service  
metadata:  
  name: kube-controllers-metrics-svc  
  namespace: kube-system  
spec:  
  clusterIP: None  
  selector:  
    k8s-app: calico-kube-controllers  
  ports:  
  \- port: 9094  
    targetPort: 9094  
EOF  
kubectl get svc,ep \-n kube-system kube-controllers-metrics-svc

  
kubectl create \-f \-<<EOF  
apiVersion: v1  
kind: Namespace  
metadata:  
  name: calico-monitoring  
  labels:  
    app:  ns-calico-monitoring  
    role: monitoring  
EOF  
kubectl get ns

  
kubectl apply \-f \- <<EOF  
apiVersion: rbac.authorization.k8s.io/v1  
kind: ClusterRole  
metadata:  
  name: calico-prometheus-user  
rules:  
\- apiGroups: \[""\]  
  resources:  
  \- endpoints  
  \- services  
  \- pods  
  verbs: \["get", "list", "watch"\]  
\- nonResourceURLs: \["/metrics"\]  
  verbs: \["get"\]  
\---  
apiVersion: v1  
kind: ServiceAccount  
metadata:  
  name: calico-prometheus-user  
  namespace: calico-monitoring  
\---  
apiVersion: rbac.authorization.k8s.io/v1  
kind: ClusterRoleBinding  
metadata:  
  name: calico-prometheus-user  
roleRef:  
  apiGroup: rbac.authorization.k8s.io  
  kind: ClusterRole  
  name: calico-prometheus-user  
subjects:  
\- kind: ServiceAccount  
  name: calico-prometheus-user  
  namespace: calico-monitoring  
EOF  
kubectl get sa \-n calico-monitoring

Prometheus metrics are enabled by default on TCP port 9094 for `calico-kube-controllers`. Created a Service to expose these metrics; and just set up a service account and appropriate permissions for Prometheus to scrape metrics.

![[Raw/Media/Resources/13793ab41d1ce0741821a538c78c950b_MD5.png]]

Create a ConfigMap with the Prometheus configuration to scrape metrics from Calico components.

kubectl apply \-f \- <<EOF  
apiVersion: v1  
kind: ConfigMap  
metadata:  
  name: prometheus-config  
  namespace: calico-monitoring  
data:  
  prometheus.yml: |-  
    global:  
      scrape\_interval:   15s  
      external\_labels:  
        monitor: 'tutorial-monitor'  
    scrape\_configs:  
    - job\_name: 'prometheus'  
      scrape\_interval: 5s  
      static\_configs:  
      - targets: \['localhost:9090'\]  
    - job\_name: 'felix\_metrics'  
      scrape\_interval: 5s  
      scheme: http  
      kubernetes\_sd\_configs:  
      - role: endpoints  
      relabel\_configs:  
      - source\_labels: \[\_\_meta\_kubernetes\_service\_name\]  
        regex: felix-metrics-svc  
        replacement: $1  
        action: keep  
    - job\_name: 'felix\_windows\_metrics'  
      scrape\_interval: 5s  
      scheme: http  
      kubernetes\_sd\_configs:  
      - role: endpoints  
      relabel\_configs:  
      - source\_labels: \[\_\_meta\_kubernetes\_service\_name\]  
        regex: felix-windows-metrics-svc  
        replacement: $1  
        action: keep  
    - job\_name: 'typha\_metrics'  
      scrape\_interval: 5s  
      scheme: http  
      kubernetes\_sd\_configs:  
      - role: endpoints  
      relabel\_configs:  
      - source\_labels: \[\_\_meta\_kubernetes\_service\_name\]  
        regex: typha-metrics-svc  
        replacement: $1  
        action: keep  
    - job\_name: 'kube\_controllers\_metrics'  
      scrape\_interval: 5s  
      scheme: http  
      kubernetes\_sd\_configs:  
      - role: endpoints  
      relabel\_configs:  
      - source\_labels: \[\_\_meta\_kubernetes\_service\_name\]  
        regex: kube-controllers-metrics-svc  
        replacement: $1  
        action: keep  
EOF  
kubectl get cm \-n calico-monitoring prometheus-config

  
kubectl apply \-f \- <<EOF  
apiVersion: v1  
kind: Pod  
metadata:  
  name: prometheus-pod  
  namespace: calico-monitoring  
  labels:  
    app: prometheus-pod  
    role: monitoring  
spec:  
  nodeSelector:  
    kubernetes.io/os: linux  
  serviceAccountName: calico-prometheus-user  
  containers:  
  \- name: prometheus-pod  
    image: prom/prometheus  
    resources:  
      limits:  
        memory: "128Mi"  
        cpu: "500m"  
    volumeMounts:  
    \- name: config-volume  
      mountPath: /etc/prometheus/prometheus.yml  
      subPath: prometheus.yml  
    ports:  
    \- containerPort: 9090  
  volumes:  
  \- name: config-volume  
    configMap:  
      name: prometheus-config  
EOF  
kubectl get pods prometheus-pod \-n calico-monitoring \-owide

View metrics

  
kubectl get pods prometheus-pod \-n calico-monitoring \-owide

  
curl <파드 IP>:9090/metrics  
curl 172.16.34.7:9090/metrics

  
kubectl apply \-f \- <<EOF  
apiVersion: v1  
kind: Service  
metadata:  
  name: prometheus-dashboard-svc  
  namespace: calico-monitoring  
spec:  
  type: NodePort  
  selector:  
    app: prometheus-pod  
    role: monitoring  
  ports:  
    \- protocol: TCP  
      port: 9090  
      targetPort: 9090  
      nodePort: 30001   
EOF  
kubectl get svc,ep \-n calico-monitoring

  
echo \-e "Prometheus URL = http://$(curl -s ipinfo.io/ip):30001"    
echo \-e "Prometheus URL = http://192.168.10.10:30001"            

![[Raw/Media/Resources/7339359dbda43ed51229eac3180c9cdf_MD5.png]]

Determine the external IP address of your cluster and access the Prometheus UI and Grafana UI using the NodePort. Open the provided URL in a web browser for Grafana. Use the default credentials to log in:

-   Username: `admin`
-   Password: `admin`

Use Grafana dashboard to view Calico component metrics.

  
kubectl apply \-f \- <<EOF  
apiVersion: v1  
kind: ConfigMap  
metadata:  
  name: grafana-config  
  namespace: calico-monitoring  
data:  
  prometheus.yaml: |-  
    {  
        "apiVersion": 1,  
        "datasources": \[  
            {  
               "access":"proxy",  
                "editable": true,  
                "name": "calico-demo-prometheus",  
                "orgId": 1,  
                "type": "prometheus",  
                "url": "http://prometheus-dashboard-svc.calico-monitoring.svc:9090",  
                "version": 1  
            }  
        \]  
    }  
EOF  
kubectl get cm \-n calico-monitoring

  
kubectl apply \-f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/grafana-dashboards.yaml

  
kubectl apply \-f \- <<EOF  
apiVersion: v1  
kind: Pod  
metadata:  
  name: grafana-pod  
  namespace: calico-monitoring  
  labels:  
    app:  grafana-pod  
    role: monitoring  
spec:  
  nodeSelector:  
    kubernetes.io/os: linux  
  containers:  
  \- name: grafana-pod  
    image: grafana/grafana:latest  
    resources:  
      limits:  
        memory: "128Mi"  
        cpu: "500m"  
    volumeMounts:  
    \- name: grafana-config-volume  
      mountPath: /etc/grafana/provisioning/datasources  
    \- name: grafana-dashboards-volume  
      mountPath: /etc/grafana/provisioning/dashboards  
    \- name: grafana-storage-volume  
      mountPath: /var/lib/grafana  
    ports:  
    \- containerPort: 3000  
  volumes:  
  \- name: grafana-storage-volume  
    emptyDir: {}  
  \- name: grafana-config-volume  
    configMap:  
      name: grafana-config  
  \- name: grafana-dashboards-volume  
    configMap:  
      name: grafana-dashboards-config  
EOF

  
kubectl get pod \-n calico-monitoring

  
kubectl apply \-f \- <<EOF  
apiVersion: v1  
kind: Service  
metadata:  
  name: grafana  
  namespace: calico-monitoring  
spec:  
  type: NodePort  
  selector:  
    app:  grafana-pod  
    role: monitoring  
  ports:  
    \- protocol: TCP  
      port: 3000  
      targetPort: 3000  
      nodePort: 30002   
EOF  
kubectl get svc,ep \-n calico-monitoring

  
echo \-e "Grafana URL = http://$(curl -s ipinfo.io/ip):30002"    
echo \-e "Grafana URL = http://192.168.10.10:30002"            

![[Raw/Media/Resources/daaaac9db3274f945ee0295639ecfcb7_MD5.png]]