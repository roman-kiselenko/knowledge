---
title: Fun DNS Facts Learned from the KIND Environment
source: https://hwchiu.medium.com/fun-dns-facts-learned-from-the-kind-environment-241e0ea8c6d4
clipped: 2023-12-21
published: 
category: k8s
tags:
  - network
read: true
---

Kubernetes in Docker (KIND) is an open-source project maintained within the Kubernetes SIG community. The purpose of this project is to provide a simple Kubernetes environment using Docker, primarily for Kubernetes CI testing.

Kubernetes itself is a container orchestration platform, so using Docker as its nodes results in an architecture based on the concept of Containers in Containers. The implementation process of this approach also introduces challenges related to the dual-layer containers. This article focuses on one specific implementation issue related to DNS that arises during this process.

The architecture of [KIND](https://github.com/kubernetes-sigs/kind?WT.mc_id=AZ-MVP-5003331) is built on top of Docker. The experimental setup for this article involves creating a three-node Kubernetes cluster using KIND on Ubuntu 20.04. One node serves as the control plane, while the other two function as general workers.

In the context of Docker, three Docker containers need to be spun up to simulate Kubernetes nodes. These containers communicate with each other using Docker Network to address network connectivity concerns. The overall architecture is depicted in the diagram below:

![[Raw/Media/Resources/163404406dd1554df5467b3edb90fd87_MD5.png]]

Docker

Within this environment, three containers will be deployed. Each container will have its own IP within the Docker Network subnet, and they will all contain the applications and libraries included in the respective container images.

For Kubernetes, in order to achieve a three-node Kubernetes cluster, there must be a control plane within the cluster that provides functionalities such as etcd, scheduler, controller, and API server. Additionally, each node needs to have kubelet and a relevant container runtime installed to manage the lifecycle of containers. The concept is illustrated in the diagram below:

![[Raw/Media/Resources/101bc922f92de5282bc9801d219e5d57_MD5.png]]

Simple Kubernetes View

The KIND environment combines these two aspects. Consequently, the overall architecture is as follows:

![[Raw/Media/Resources/b1ee14fff9830b997b661e77760ad70d_MD5.png]]

How Kubernetes Works in KIND

Containers launched through Docker will have Containerd installed to manage the lifecycle of Kubernetes containers. Simultaneously, connections with the control plane will be established through components like kubelet, forming the Kubernetes cluster.

Readers familiar with Docker Compose are likely aware that for convenient communication between containers, container names can be used directly as DNS targets. This design eliminates the need for containers to worry about IP changes. Docker, in fact, embeds a DNS server within its system to handle this issue, with the DNS server’s fixed IP being 127.0.0.11.

The responsibilities of this DNS server can be categorized as follows:

1.  If the DNS request is for a container name, the IP of the container is returned.
2.  Otherwise, based on the host configuration, the DNS request is forwarded to an upstream DNS server.

In the example depicted below, two containers, named “hwchiu” and “hwchiu2,” are running. Using nslookup, one can easily resolve the corresponding IP addresses. It’s also observable that the **/etc/hosts** file within these containers dynamically point to 127.0.0.11. This implies that all DNS requests within the containers are redirected to the built-in Docker DNS server.

![[Raw/Media/Resources/a29c7b798948a06c9bae310c493bb318_MD5.png]]

Docker DNS Example

However, Docker’s DNS implementation raises some questions that are worth exploring:

1.  Where exactly does the Docker DNS server run? When does Docker start running? Why can’t its traces be found within a container through commands like **ps**?
2.  Common DNS communication usually takes place on port 53. If the Docker DNS server occupies port 53, does it prevent containers from running a second service that uses port 53? Does this complexity make it challenging to deploy other DNS server services within Docker containers?

To address these uncertainties, it’s necessary to understand the implementation of Docker DNS. By gaining insights into its implementation, we can provide precise answers to the questions mentioned above.

## Implementation

The design of Docker DNS is remarkably clever, using the concept of Linux namespaces to address this issue seamlessly. It enables all Docker DNS servers to run on the host itself (within the PID namespace), while the networking aspect listens within each container (within the network namespace).

This architecture allows you to access the DNS server through 127.0.0.11, but you cannot find the process of this DNS server within the container.

Furthermore, to prevent conflicts between the Docker DNS server and user-defined DNS services, Docker DNS avoids using Port 53 and instead adopts a random port number.

The overall architecture is depicted in the following diagram:

![[Raw/Media/Resources/efa7ceb06f383a233a8e4bcee33542a2_MD5.png]]

Docker DNS View

However, for container services, as **/etc/hosts** has been altered to use 127.0.0.11 as the default DNS search and typically relies on Port 53, Dockerd relies on **iptables** to dynamically adjust rules. This involves modifying the target port of all packets sent to 127.0.0.11:53 to handle the connection issue.

Therefore, if you observe from within a container using commands like nsenter, using **ss** and i**ptables** commands, you will see results as shown in the diagram below:

![[Raw/Media/Resources/3ecc3aa81d75c58d9219e773447ae838_MD5.png]]

In this display, **ss** shows that within the environment, 127.0.0.11 is listening on two ports, corresponding to TCP and UDP DNS requests, and the process is **dockerd**. **iptables** reveals the relevant DNAT rules.

With these mechanisms in place, all DNS requests from containers are processed by the Docker DNS, without preemptively occupying Port 53.

Here you can find the relevant [source code](https://github.com/moby/libnetwork/blob/67e0588f1ddfaf2faf4c8cae8b7ea2876434d91c/resolver_unix.go?WT.mc_id=AZ-MVP-5003331), which demonstrates that Docker dynamically modifies four rules to address DNAT + SNAT requirements.

...  
 resolverIP, ipPort, \_ := net.SplitHostPort(os.Args\[2\])  
 \_, tcpPort, \_ := net.SplitHostPort(os.Args\[3\])  
 rules := \[\]\[\]string{  
  {"-t", "nat", "-I", outputChain, "-d", resolverIP, "-p", "udp", "--dport", dnsPort, "-j", "DNAT", "--to-destination", os.Args\[2\]},  
  {"-t", "nat", "-I", postroutingchain, "-s", resolverIP, "-p", "udp", "--sport", ipPort, "-j", "SNAT", "--to-source", ":" + dnsPort},  
  {"-t", "nat", "-I", outputChain, "-d", resolverIP, "-p", "tcp", "--dport", dnsPort, "-j", "DNAT", "--to-destination", os.Args\[3\]},  
  {"-t", "nat", "-I", postroutingchain, "-s", resolverIP, "-p", "tcp", "--sport", tcpPort, "-j", "SNAT", "--to-source", ":" + dnsPort},  
 }  
...

Up to this point, you should now have a basic understanding of Docker DNS. Moving forward, let’s take a look at the situation with Kubernetes.

In a Kubernetes cluster, there is also a DNS server — from the early Kube-DNS to the present CoreDNS. Its functions are quite similar to Docker DNS:

1.  If the DNS request pertains to an internal Kubernetes service, it responds with internal information.
2.  Otherwise, it forwards the request to an upstream DNS server.

Similar to Docker DNS, both aim to handle service-specific requests and forward if necessary.

For CoreDNS, the upstream DNS server, by default, is the DNS server used by the node, which is the same as the previously mentioned 127.0.0.11.

The process unfolds as follows:

![[Raw/Media/Resources/aa720ab084a6c3058343f848b65c27da_MD5.png]]

How CoreDNS Handle DNS

Suppose a Pod on “worker2” wants to make a DNS request. This request is directed to CoreDNS. When CoreDNS can’t resolve it and tries to forward it to an upstream DNS server, it ends up forwarding to 127.0.0.11, which is where the issue arises.

## Issue

Because 127.0.0.11 is specific to Dockerd, the CoreDNS managed by Containerd on Docker containers naturally lacks this functionality. It doesn’t possess the relevant iptables and Docker DNS server.

To resolve this, KIND’s approach is to prevent CoreDNS from sending packets to 127.0.0.11. Instead, CoreDNS sends them to the node’s IP, which then forwards the packets to the 127.0.0.11 service on the node.

The process is outlined as follows:

![[Raw/Media/Resources/0dfd7606ec3b17a4c30d91c667dfd763_MD5.png]]

Idea To Fix the CoreDNS Issue

As previously mentioned, CoreDNS itself uses the node’s **/etc/hosts** as the upstream server. The approach here is to dynamically modify the node’s **/etc/hosts**, changing the default DNS server from 127.0.0.11 to the node’s own IP, such as 172.18.0.2 in the example diagram.

Once the default request location is altered, the iptables rules set up by Dockerd no longer apply. This necessitates the adjustment of iptables rules again.

The initial iptables rules set up by Dockerd are as follows:

![[Raw/Media/Resources/298ab2d852618839300c193aa963880a_MD5.png]]

iptables before KIND changes

However, KIND modifies them to the following (example from different nodes, with node IP being 172.18.0.1):

![[Raw/Media/Resources/65100726e41e703ee7489cfbbb4b3e89_MD5.png]]

iptables after KIND changes

Now, all packets sent to **172.18.0.1:53** will be redirected to **127.0.0.11:33501/41285**, the Docker DNS location. This also involves changes to SNAT.

From the [KIND source code](https://github.com/kubernetes-sigs/kind/blob/main/images/base/files/usr/local/bin/entrypoint#L459-L470?WT.mc_id=AZ-MVP-5003331), it can be observed that KIND’s K8s nodes modify iptables rules every time they start and also update the default address in /etc/resolve.conf.

  
local docker\_embedded\_dns\_ip='127.0.0.11'

  
local docker\_host\_ip  
docker\_host\_ip="$( (head -n1 <(timeout 5 getent ahostsv4 'host.docker.internal') | cut -d' ' -f1) || true)"  
  
if \[\[ -z "${docker\_host\_ip}" \]\] || \[\[ $docker\_host\_ip =~ ^127\\.\[0-9\]+\\.\[0-9\]+\\.\[0-9\]+$ \]\]; then  
  docker\_host\_ip=$(ip -4 route show default | cut -d' ' -f3)  
fi

  
iptables-save \\  
  | sed \\  
    \`  
    -e "s/-d ${docker\_embedded\_dns\_ip}/-d ${docker\_host\_ip}/g" \\  
    \`  
    -e 's/-A OUTPUT \\(.\*\\) -j DOCKER\_OUTPUT/\\0\\n-A PREROUTING \\1 -j DOCKER\_OUTPUT/' \\  
    \`  
    -e "s/--to-source :53/--to-source ${docker\_host\_ip}:53/g"\\  
    \`  
    \`  
    -e "s/p -j DNAT --to-destination ${docker\_embedded\_dns\_ip}/p --dport 53 -j DNAT --to-destination ${docker\_embedded\_dns\_ip}/g" \\  
  | iptables-restore

  
cp /etc/resolv.conf /etc/resolv.conf.original  
replaced="$(sed -e "s/${docker\_embedded\_dns\_ip}/${docker\_host\_ip}/g" /etc/resolv.conf.original)"  
if \[\[ "${KIND\_DNS\_SEARCH+x}" == "" \]\]; then  
    
  echo "$replaced" >/etc/resolv.conf  
elif \[\[ -z "$KIND\_DNS\_SEARCH" \]\]; then  
    
  echo "$replaced" | grep -v "^search" >/etc/resolv.conf  
else  
    
  {  
    echo "search $KIND\_DNS\_SEARCH";  
    echo "$replaced" | grep -v "^search";  
  } >/etc/resolv.conf  
fi

With these adjustments, CoreDNS sends DNS requests to the node, which are then forwarded to Docker DNS through iptables. If Docker DNS cannot handle them, the requests are forwarded upwards to the initial settings on the host, creating a cascading process.

1.  Docker has an integrated DNS server to handle DNS requests between Docker containers.
2.  Docker simplifies the deployment and operation of DNS through iptables and namespaces.
3.  Kubernetes’ CoreDNS is influenced by Docker DNS by default due to /etc/resolve modifications.
4.  KIND modifies these rules twice to correct all routes, ensuring DNS forwarding.