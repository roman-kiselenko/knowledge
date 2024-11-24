---
title: Learning Kubernetes VXLAN Network with Flannel
source: https://addozhang.medium.com/learning-kubernetes-vxlan-network-with-flannel-2d6a58c95300
clipped: 2024-02-05
published: 
category: network
tags:
  - k8s
read: false
---

Flannel serves as a remarkably straightforward overlay network (VXLAN), standing as one of the solutions for Kubernetes Network CNI. This system operates by deploying a streamlined lightweight agent, `flanneld`, on each host to monitor node variations within the cluster and pre-configure address spaces accordingly. Additionally, Flannel establishes VTEP `flannel.1` (VXLAN tunnel endpoints) on every host, facilitating connections with other hosts through VXLAN tunnels.

The `flanneld` is poised to listen on port `8472`, orchestrating data transfer with the VTEPs of other nodes via UDP. Packets reaching the VTEP at layer two are conveyed intact through UDP, transmitted to the VTEP at the counterpart end, followed by unpacking and processing at layer two. Essentially, it harnesses the fourth-layer UDP to convey second-layer data frames.

Within the Kubernetes distribution, [K3S](https://k3s.io/) incorporates Flannel as the default CNI implementation. K3S integrates seamlessly with flannel, operating through a go-routine post-initiation.

While utilizing the K3S distribution for the Kubernetes cluster, it is imperative to disable the integrated flannel during installation and validate using an independently installed flannel. This is necessitated by the location of the CNI bin directory in k3s at `/var/lib/rancher/k3s/data/xxx/bin` rather than the `/opt/cni/bin`.

To install the CNI plugin, the following command needs to be executed on all node instances to facilitate the official CNI bin download.

```bash
sudo mkdir -p /opt/cni/bin  
curl -sSL https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz | sudo tar -zxf - -C /opt/cni/bin
```

Install the control plane of k3s.

```
export INSTALL\_K3S\_VERSION=v1.23.8+k3s2  
curl -sfL https://get.k3s.io | sh -s - --disable traefik --flannel-backend=none --write-kubeconfig-mode 644 --write-kubeconfig ~/.kube/config
```

Install Flannel. **It is pertinent to note that the default Pod CIDR for Flannel is** `**10.244.0.0/16**`**; however, we will modify this to align with the k3s default of** `**10.42.0.0/16**`**.**

```
curl -s https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml | sed 's|10.244.0.0/16|10.42.0.0/16|g' | kubectl apply -f -
```

Add an additional node into the cluster.

```
export INSTALL\_K3S\_VERSION=v1.23.8+k3s2  
export MASTER\_IP=<MASTER\_IP>  
export NODE\_TOKEN=<TOKEN>  
curl -sfL https://get.k3s.io | K3S\_URL=https://${MASTER\_IP}:6443 K3S\_TOKEN=${NODE\_TOKEN} sh -
```

Examine the status of the nodes.

```
kubectl get node  
NAME          STATUS   ROLES                  AGE   VERSION  
ubuntu\-dev3   Ready    <none\>                 13m   v1.23.8+k3s2  
ubuntu\-dev2   Ready    control\-plane,master   17m   v1.23.8+k3s2
```

Initiate two pods: `curl` and `httpbin`.

```
NODE1=ubuntu-dev2  
NODE2=ubuntu-dev3  
kubectl apply \-n default \-f \- <<EOF  
apiVersion: v1  
kind: Pod  
metadata:  
  labels:  
    app: curl  
  name: curl  
spec:  
  containers:  
  \- image: curlimages/curl  
    name: curl  
    command: \["sleep", "365d"\]  
  nodeName: $NODE1  
\---  
apiVersion: v1  
kind: Pod  
metadata:  
  labels:  
    app: httpbin  
  name: httpbin  
spec:  
  containers:  
  \- image: kennethreitz/httpbin  
    name: httpbin  
  nodeName: $NODE2  
EOF
```

Next, let’s delve into how the CNI plugin configures the pod network.

Flannel is deployed via a `Daemonset`, ensuring a flannel pod operates on each node. Leveraging local disk mounting, the initialization container copies the binary files and CNI configurations to the local disk during Pod initiation, situated respectively at `/opt/cni/bin/flannel` and `/etc/cni/net.d/10-flannel.conflist`.

By examining the `ConfigMap` in the [kube-flannel.yml](https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml), one can pinpoint the CNI configuration. Flannel, by default, delegates (see [flannel-cni source code](https://github.com/flannel-io/cni-plugin/blob/v1.1.0/flannel_linux.go#L78) [flannel_linux.go#L78](https://github.com/flannel-io/cni-plugin/blob/v1.1.0/flannel_linux.go#L78)) the network configuration tasks to the [bridge plugin](https://www.cni.dev/plugins/current/main/bridge/), establishing a network named `cbr0`. Meanwhile, the IP address management is entrusted (see [flannel-cni source code](https://github.com/flannel-io/cni-plugin/blob/v1.1.0/flannel_linux.go#L40) [flannel_linux.go#L40](https://github.com/flannel-io/cni-plugin/blob/v1.1.0/flannel_linux.go#L40)) to the [host-local plugin](https://www.cni.dev/plugins/current/ipam/host-local/) for execution.

```
{  
  "name": "cbr0",  
  "cniVersion": "0.3.1",  
  "plugins": \[  
    {  
      "type": "flannel",  
      "delegate": {  
        "hairpinMode": true,  
        "isDefaultGateway": true  
      }  
    },  
    {  
      "type": "portmap",  
      "capabilities": {  
        "portMappings": true  
      }  
    }  
  \]  
}
```

Furthermore, Flannel’s network configuration encompasses the preset Pod CIDR `10.42.0.0/16` and the backend type `vxlan`, which stands as Flannel's default type. Additionally, several other [backend types are available](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md) for selection, such as `host-gw`, `wireguard`, `udp`, `Alloc`, `IPIP`, and `IPSec`.

```
  
{  
  "Network": "10.42.0.0/16",  
  "Backend": {  
    "Type": "vxlan"  
  }  
}
```

Upon initiation, the Flannel Pod activates the `flanneld` process, stipulating parameters `--ip-masq` and `--kube-subnet-mgr`, with the latter initiating the `kube subnet manager` mode.

![[Raw/Media/Resources/119a9087d367807415b43de8ada7ca4e_MD5.png]]

During cluster initialization, the default Pod CIDR `10.42.0.0/16` is utilized. As nodes join the cluster, the cluster allocates **node-specific Pod CIDR** `10.42.X.1/24` from this segment.

In the `kube subnet manager` mode, flannel connects to the apiserver, monitoring node update events and retrieving the Pod CIDR from node information.

```
kubectl get no ubuntu-dev2 -o jsonpath={.spec} | jq  
{  
  "podCIDR": "10.42.0.0/24",  
  "podCIDRs": \[  
    "10.42.0.0/24"  
  \],  
  "providerID": "k3s://ubuntu-dev2"  
}
```

Subsequently, a subnet configuration file is crafted on the host, showcasing the contents of one of the node’s subnet configuration files herein. The content variation in another node lies in `FLANNEL_SUBNET=10.42.1.1/24`, adopting the corresponding node's Pod CIDR.

  
```
cat /run/flannel/subnet.env  
FLANNEL\_NETWORK=10.42.0.0/16  
FLANNEL\_SUBNET=10.42.0.1/24  
FLANNEL\_MTU=1450  
FLANNEL\_IPMASQ=true
```

The execution of the CNI plugin is precipitated by the container runtime, for which specifics can be gleaned from the preceding article, [Source Code Analysis: Examining the Utilization of CNI Through kubelet and Container Runtime](https://medium.com/@addozhang/source-code-analysis-understanding-cnis-usage-from-kubelet-container-runtime-24d72f29466b).

![[Raw/Media/Resources/d0be44048c20965c407bde9047647d85_MD5.png]]

## Flannel Plugin

During the execution phase, the `flannel` CNI plugin (`/opt/cni/bin/flannel`) accepts the incoming `cni-conf.json`, reads the previously initialized `subnet.env` configurations, outputs the results, and delegates further steps to the `bridge`.

```
cat /var/lib/cni/flannel/e4239ab2706ed9191543a5c7f1ef06fc1f0a56346b0c3f2c742d52607ea271f0 | jq  
{  
  "cniVersion": "0.3.1",  
  "hairpinMode": true,  
  "ipMasq": false,  
  "ipam": {  
    "ranges": \[  
      \[  
        {  
          "subnet": "10.42.0.0/24"  
        }  
      \]  
    \],  
    "routes": \[  
      {  
        "dst": "10.42.0.0/16"  
      }  
    \],  
    "type": "host-local"  
  },  
  "isDefaultGateway": true,  
  "isGateway": true,  
  "mtu": 1450,  
  "name": "cbr0",  
  "type": "bridge"  
}
```

## Bridge Plugin

Utilizing the aforementioned output and parameters as input, the `bridge` undertakes the following operations based on the configurations:

1.  Establishes bridge `cni0` (within the node's root network namespace)
2.  Constructs container network interface `eth0` (within the pod network namespace)
3.  Generates virtual network interface `vethX` on the host (within the node's root network namespace)
4.  Connects `vethX` to the bridge `cni0`
5.  Delegates IP address allocation, DNS, and routing tasks to the ipam plugin
6.  Binds the IP address to the interface `eth0` within the pod network namespace
7.  Inspects the state of the bridge
8.  Orchestrates routing setup
9.  Configures DNS

Eventually, the following results are manifested:

```
cat /var/li/cni/results/cbr0-a34bb3dc268e99e6e1ef83c732f5619ca89924b646766d1ef352de90dbd1c750-eth0 | jq .result  
{  
  "cniVersion": "0.3.1",  
  "dns": {},  
  "interfaces": \[  
    {  
      "mac": "6a:0f:94:28:9b:e7",  
      "name": "cni0"  
    },  
    {  
      "mac": "ca:b4:a9:83:0f:d4",  
      "name": "veth38b50fb4"  
    },  
    {  
      "mac": "0a:01:c5:6f:57:67",  
      "name": "eth0",  
      "sandbox": "/var/run/netns/cni-44bb41bd-7c41-4860-3c55-4323bc279628"  
    }  
  \],  
  "ips": \[  
    {  
      "address": "10.42.0.5/24",  
      "gateway": "10.42.0.1",  
      "interface": 2,  
      "version": "4"  
    }  
  \],  
  "routes": \[  
    {  
      "dst": "10.42.0.0/16"  
    },  
    {  
      "dst": "0.0.0.0/0",  
      "gw": "10.42.0.1"  
    }  
  \]  
}
```

## Port-mapping Plugin

This plugin facilitates the forwarding of traffic from one or multiple ports on the host to the container.

Let us embark on packet capture using `tcpdump` on the interface `cni0` at the initial node.

```
tcpdump -i cni0 port 80 -vvv
```

Commence a request from the pod `curl`, utilizing the IP address `10.42.1.2` of pod `httpbin`:

```
kubectl exec curl \-n default 
```

As per the packet capture results on `cni0`, the third-layer IP addresses are all pod IP addresses, giving an impression of both pods residing within the same network segment.

![[Raw/Media/Resources/6b71b567c8db45e2a84b04c77496b1ba_MD5.png]]

At the outset of this article, it was noted that `flanneld` monitors the UDP port 8472.

```
netstat -tupln | grep 8472  
udp        0      0 0.0.0.0:8472            0.0.0.0:\*                           -
```

We can capture UDP packets directly on the ethernet interface:

```
tcpdump -i eth0 port 8472 -vvv
```

Upon resending the request, it is observable that the UDP packets are captured, wherein the transmitted payload is encapsulated at the second layer (layer 2).

![[Raw/Media/Resources/2f3343c4744998c64ddae01028946819_MD5.png]]

In the first article of this series, we explored the communication between pods, mentioning the various handling methods adopted by different CNI plugins. This time, we delve into the functioning principles of the flannel plugin. Hopefully, the following diagram will offer a more intuitive understanding of how the overlay network manages cross-node network communication.

![[Raw/Media/Resources/c7e0e67bef4c56c5b07e6b0b0700d201_MD5.gif]]

When traffic addressed to `10.42.1.2` reaches the bridge `cni0` at node A, given that the target IP does not belong to the current segment of the network, it follows system routing rules and proceeds to the interface `flannel.1`, or what is known as the VXLAN's vtep (VXLAN Tunnel Endpoint). The routing rules here are maintained by `flanneld`, which updates the rules whenever a node goes online or offline.

```
#192.168.1.12  
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface  
default         \_gateway        0.0.0.0         UG    0      0        0 eth0  
10.42.0.0       0.0.0.0         255.255.255.0   U     0      0        0 cni0  
10.42.1.0       10.42.1.0       255.255.255.0   UG    0      0        0 flannel.1  
192.168.1.0     0.0.0.0         255.255.255.0   U     0      0        0 eth0  
#192.168.1.13  
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface  
default         \_gateway        0.0.0.0         UG    0      0        0 eth0  
10.42.0.0       10.42.0.0       255.255.255.0   UG    0      0        0 flannel.1  
10.42.1.0       0.0.0.0         255.255.255.0   U     0      0        0 cni0  
192.168.1.0     0.0.0.0         255.255.255.0   U     0      0        0 eth0
```

`flannel.1` re-encapsulates the original ethernet packet using the UDP protocol and forwards it to the target address `10.42.1.0` (the target's MAC address is acquired through ARP). The counterpart vtep, which is also referred to as the UDP port 8472 of `flannel.1`, receives the message, decapsulates the ethernet packet, and directs the ethernet packet through routing processes, eventually delivering it to the interface `cni0`, and subsequently to the targeted pod.

The data transmission process for responses mirrors that of requests, with the only difference being that the source and destination addresses are switched.