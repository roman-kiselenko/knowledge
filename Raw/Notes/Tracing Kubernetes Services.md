---
title: Tracing Kubernetes Services
source: https://itnext.io/tracing-kubernetes-services-4dc827abdc55
clipped: 2024-12-23
published: 
category: network
tags:
  - network
read: false
---

TL;DR — Iptables is very brain hurty (I hope this is understood as a warning for what the rest of this post will cover…)

If you have messed around with Kubernetes at all, you have undoubtedly had to deploy or interact with a service at some point. How do services work, though? This post will peak under the hood to see how services work. For a great read on Kubernetes services in general and why they’re needed, see [this post](https://medium.com/@betz.mark/understanding-kubernetes-networking-services-f0cb48e4cc82) from @mark.betz.

![[Raw/Media/Resources/e8a8505e94baaa39398009dea862c1a7_MD5.png]]

Here is a quick description of the environment:

-   RKE2 cluster running 1.28.4+rk2r1
-   Calico CNI running v3.26.3 using a VxLAN overlay
-   All nodes are running Ubuntu 22.04

Before we can dig into the guts of a service, we need a workload and service. The simple [Podinfo](https://github.com/stefanprodan/podinfo) web app will be used for this purpose.

```
$ helm repo add podinfo https://stefanprodan.github.io/podinfo  
"podinfo" already exists with the same configuration, skipping
```

  
```
$ helm repo update  
Hang tight while we grab the latest from your chart repositories...  
...Successfully got an update from the "podinfo" chart repository  
...Successfully got an update from the "haproxytech" chart repository  
Update Complete. ⎈Happy Helming!⎈
```

  
```
$ helm upgrade -i --install --wait frontend --namespace podinfo \\  
\--set replicaCount=2 \\  
\--set backend=http://backend-podinfo:9898/echo \\  
podinfo/podinfo  
Release "frontend" does not exist. Installing it now.  
NAME: frontend  
LAST DEPLOYED: Wed Mar 13 15:41:29 2024  
NAMESPACE: podinfo  
STATUS: deployed  
REVISION: 1  
NOTES:  
1\. Get the application URL by running these commands:  
  echo "Visit http://127.0.0.1:8080 to use your application"  
  kubectl -n podinfo port-forward deploy/frontend-podinfo 8080:9898
```

Let’s see where this workload got scheduled:

And let’s see what IP address was assigned to the service that was spun up:

![[Raw/Media/Resources/ce9d145c17b5e532f14943e6631a7480_MD5.png]]

A quick sidebar is needed here to explain why the rest of this post will focus on iptables. As Mark points out in his services post, iptables is the user space program that fronts netfilter, a data plane packet processing engine that can, among other things, redirect traffic to another destination.

Iptables is the default program that underpins kube-proxy in most Kubernetes distributions. A list of the other programs that kube-proxy can use is found [here](https://kubernetes.io/docs/reference/networking/virtual-ips/#proxy-modes). This post is only valid for iptables based installations.

This next section will be run from a cluster node that did not have a Podinfo workload scheduled to it. Let’s enable tracing in iptables for traffic that hits the destination port of 9898, the service port for Podinfo.

$ iptables -t raw -A PREROUTING -p tcp --dport 9898 -j TRACE  
$ iptables -t raw -A OUTPUT -p tcp --dport 9898 -j TRACE

> Note that since the above filter only captures a destination port of 9898, this filter will not catch return traffic from the Podinfo service.

The following command was run and left running from a second terminal session with root privileges.

$ xtables-monitor --trace

Returning to the first terminal session, let’s curl against the Podinfo service:

$ curl http://10.43.156.98:9898  
{  
  "hostname": "frontend-podinfo-7854c7cd4c-jvdx5",  
  "version": "6.6.0",  
  "revision": "357009a86331a987811fefc11be1350058da33fc",  
  "color": "#34577c",  
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle\_clap.gif",  
  "message": "greetings from podinfo v6.6.0",  
  "goos": "linux",  
  "goarch": "amd64",  
  "runtime": "go1.21.7",  
  "num\_goroutine": "8",  
  "num\_cpu": "8"  
}

Before proceeding any further, disable the packet trace, as this can log an excessive amount of data on busy systems:

$ iptables -t raw -D PREROUTING -p tcp --dport 9898 -j TRACE  
$ iptables -t raw -D OUTPUT -p tcp --dport 9898 -j TRACE

Jumping back to the terminal where the `xtables-monitor` command was running, there was a [ton of output](https://raw.githubusercontent.com/TheFutonEng/k8s-svc-trace/main/trace.txt). Let’s first talk about some columns in this mess:

PACKET: 2 0427ffe5 OUT=enp3s0 SRC=192.168.1.88 DST=10.43.156.98 LEN=60 TOS=0x0 TTL=64 ID=45471DF SPORT=49058 DPORT=9898 SYN  
 TRACE: 2 0427ffe5 raw:OUTPUT:rule:0x18:CONTINUE  -4 -t raw -A OUTPUT -p tcp -m tcp --dport 9898 -j TRACE  
 TRACE: 2 0427ffe5 raw:OUTPUT:return:  
 TRACE: 2 0427ffe5 raw:OUTPUT:policy:ACCEPT  
 TRACE: 2 0427ffe5 mangle:OUTPUT:return:  
 TRACE: 2 0427ffe5 mangle:OUTPUT:policy:ACCEPT  
PACKET: 2 0427ffe5 OUT=enp3s0 SRC=192.168.1.88 DST=10.43.156.98 LEN=60 TOS=0x0 TTL=64 ID=45471DF SPORT=49058 DPORT=9898 SYN  
 TRACE: 2 0427ffe5 nat:OUTPUT:rule:0xb9:JUMP:cali-OUTPUT  -4 -t nat -A OUTPUT -m comment --comment "cali:tVnHkvAo15HuiPy0" -j cali-OUTPUT  
 TRACE: 2 0427ffe5 nat:cali-OUTPUT:rule:0xb8:JUMP:cali-fip-dnat  -4 -t nat -A cali-OUTPUT -m comment --comment "cali:GBTAv2p5CwevEyJm" -j cali-fip-dnat  
 TRACE: 2 0427ffe5 nat:cali-fip-dnat:return:  
 TRACE: 2 0427ffe5 nat:cali-OUTPUT:return:  
 TRACE: 2 0427ffe5 nat:OUTPUT:rule:0x9:JUMP:KUBE-SERVICES  -4 -t nat -A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES  
 TRACE: 2 0427ffe5 nat:KUBE-SERVICES:rule:0x4aaa:JUMP:KUBE-SVC-Y4T5L63IYP3YFEBS  -4 -t nat -A KUBE-SERVICES -d 10.43.156.98/32 -p tcp -m comment --comment "podinfo/frontend-podinfo:http cluster IP" -j KUBE-SVC-Y4T5L63IYP3YFEBS  
 TRACE: 2 0427ffe5 nat:KUBE-SVC-Y4T5L63IYP3YFEBS:rule:0x471b:JUMP:KUBE-MARK-MASQ  -4 -t nat -A KUBE-SVC-Y4T5L63IYP3YFEBS ! -s 10.42.0.0/16 -d 10.43.156.98/32 -p tcp -m comment --comment "podinfo/frontend-podinfo:http cluster IP" -j KUBE-MARK-MASQ  
 TRACE: 2 0427ffe5 nat:KUBE-MARK-MASQ:rule:0x4aa4:CONTINUE  -4 -t nat -A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000  
 TRACE: 2 0427ffe5 nat:KUBE-MARK-MASQ:return:  
 TRACE: 2 0427ffe5 nat:KUBE-SVC-Y4T5L63IYP3YFEBS:rule:0x471d:JUMP:KUBE-SEP-OYNVDBAPYJ623W6H  -4 -t nat -A KUBE-SVC-Y4T5L63IYP3YFEBS -m comment --comment "podinfo/frontend-podinfo:http -> 10.42.54.194:9898" -j KUBE-SEP-OYNVDBAPYJ623W6H  
 TRACE: 2 0427ffe5 nat:KUBE-SEP-OYNVDBAPYJ623W6H:rule:0x4721:ACCEPT  -4 -t nat -A KUBE-SEP-OYNVDBAPYJ623W6H -p tcp -m comment --comment "podinfo/frontend-podinfo:http" -m tcp -j DNAT --to-destination 10.42.54.194:9898  
PACKET: 2 0427ffe5 OUT=enp3s0 SRC=192.168.1.88 DST=10.42.54.194 LEN=60 TOS=0x0 TTL=64 ID=45471DF SPORT=49058 DPORT=9898 SYN MARK=0x4000  
 TRACE: 2 0427ffe5 filter:OUTPUT:rule:0x91:JUMP:cali-OUTPUT  -4 -t filter -A OUTPUT -m comment --comment "cali:tVnHkvAo15HuiPy0" -j cali-OUTPUT  
 TRACE: 2 0427ffe5 filter:cali-OUTPUT:rule:0x8a:CONTINUE  -4 -t filter -A cali-OUTPUT -m comment --comment "cali:iC1pSPgbvgQzkUk\_" -j MARK --set-xmark 0x0/0xf0000  
 TRACE: 2 0427ffe5 filter:cali-OUTPUT:return:  
 TRACE: 2 0427ffe5 filter:OUTPUT:rule:0x17:JUMP:KUBE-PROXY-FIREWALL  -4 -t filter -A OUTPUT -m conntrack --ctstate NEW -m comment --comment "kubernetes load balancer firewall" -j KUBE-PROXY-FIREWALL  
 TRACE: 2 0427ffe5 filter:KUBE-PROXY-FIREWALL:return:  
 TRACE: 2 0427ffe5 filter:OUTPUT:rule:0x12:JUMP:KUBE-SERVICES  -4 -t filter -A OUTPUT -m conntrack --ctstate NEW -m comment --comment "kubernetes service portals" -j KUBE-SERVICES  
 TRACE: 2 0427ffe5 filter:KUBE-SERVICES:return:  
 TRACE: 2 0427ffe5 filter:OUTPUT:rule:0x6:JUMP:KUBE-FIREWALL  -4 -t filter -A OUTPUT -j KUBE-FIREWALL  
 TRACE: 2 0427ffe5 filter:KUBE-FIREWALL:return:  
 TRACE: 2 0427ffe5 filter:OUTPUT:return:  
 TRACE: 2 0427ffe5 filter:OUTPUT:policy:ACCEPT  
PACKET: 2 0427ffe5 OUT=vxlan.calico SRC=192.168.1.88 DST=10.42.54.194 LEN=60 TOS=0x0 TTL=64 ID=45471DF SPORT=49058 DPORT=9898 SYN MARK=0x4000  
 TRACE: 2 0427ffe5 mangle:POSTROUTING:rule:0x15:JUMP:cali-POSTROUTING  -4 -t mangle -A POSTROUTING -m comment --comment "cali:O3lYWMrLQYEMJtB5" -j cali-POSTROUTING  
 TRACE: 2 0427ffe5 mangle:cali-POSTROUTING:rule:0x12:CONTINUE  -4 -t mangle -A cali-POSTROUTING -m comment --comment "cali:nnqPh8lh2VOogSzX" -j MARK --set-xmark 0x0/0xf0000  
 TRACE: 2 0427ffe5 mangle:cali-POSTROUTING:rule:0x13:JUMP:cali-to-host-endpoint  -4 -t mangle -A cali-POSTROUTING -m comment --comment "cali:nquN8Jw8Tz72pcBW" -m conntrack --ctstate DNAT -j cali-to-host-endpoint  
 TRACE: 2 0427ffe5 mangle:cali-to-host-endpoint:return:  
 TRACE: 2 0427ffe5 mangle:cali-POSTROUTING:return:  
 TRACE: 2 0427ffe5 mangle:POSTROUTING:return:  
 TRACE: 2 0427ffe5 mangle:POSTROUTING:policy:ACCEPT  
PACKET: 2 0427ffe5 OUT=vxlan.calico SRC=192.168.1.88 DST=10.42.54.194 LEN=60 TOS=0x0 TTL=64 ID=45471DF SPORT=49058 DPORT=9898 SYN MARK=0x4000  
 TRACE: 2 0427ffe5 nat:POSTROUTING:rule:0xba:JUMP:cali-POSTROUTING  -4 -t nat -A POSTROUTING -m comment --comment "cali:O3lYWMrLQYEMJtB5" -j cali-POSTROUTING  
 TRACE: 2 0427ffe5 nat:cali-POSTROUTING:rule:0xb5:JUMP:cali-fip-snat  -4 -t nat -A cali-POSTROUTING -m comment --comment "cali:Z-c7XtVd2Bq7s\_hA" -j cali-fip-snat  
 TRACE: 2 0427ffe5 nat:cali-fip-snat:return:  
 TRACE: 2 0427ffe5 nat:cali-POSTROUTING:rule:0xb6:JUMP:cali-nat-outgoing  -4 -t nat -A cali-POSTROUTING -m comment --comment "cali:nYKhEzDlr11Jccal" -j cali-nat-outgoing  
 TRACE: 2 0427ffe5 nat:cali-nat-outgoing:return:  
 TRACE: 2 0427ffe5 nat:cali-POSTROUTING:rule:0xb7:ACCEPT  -4 -t nat -A cali-POSTROUTING -o vxlan.calico -m comment --comment "cali:e9dnSgSVNmIcpVhP" -m addrtype ! --src-type LOCAL --limit-iface-out -m addrtype --src-type LOCAL -j MASQUERADE --random-fully

-   **PACKET:** This indicates the beginning of a new packet event. It signifies that the line contains information about a packet that is being processed, including its source, destination, length, and other packet-level details. The `PACKET:` lines also contain the outgoing interface based on the current destination IP address of the packet.
-   **TRACE:** This indicates a tracing line that follows a packet's path through the various iptables chains and rules. Each “TRACE” line represents a step in the packet's processing, showing which rule or chain the packet is currently being evaluated against.
-   **Protocol Number:** This is the `2`after `TRACE:`or `PACKET:`, and it represents AF\_INET in this case (IPv4 traffic).
-   **Packet Identifier:** The value `0427ffe5` in the above output serves as a unique packet identifier, allowing for the correlation of trace messages of a specific packet.
-   **Table:Chain:** for `TRACE` entries, the next column refers to the iptables table and chain that is processing the packet. In the case of the second packet, `raw:OUTPUT` refers to the `raw` table and the `OUTPUT` chain. To show just that iptables chain, run `sudo iptables -t raw -L OUTPUT -n --line-numbers`

One other important piece of data is understanding how netfilter processes packets. For a packet originating on a system, the order is:

1.  Routing
2.  Raw (OUTPUT)
3.  Mangle (OUTPUT)
4.  NAT (OUTPUT)
5.  Filter (OUTPUT)
6.  Routing
7.  Mangle (POSTROUTING)
8.  NAT (POSTROUTING)

More details can be found [here](https://rlworkman.net/howtos/iptables/chunkyhtml/c962.html), specifically in section 6–2. The below diagram from [Wikipedia](https://en.wikipedia.org/wiki/Netfilter#/media/File:Netfilter-packet-flow.svg) is also a great reference to show the packet processing order.

To be more explicit, this is the section of the diagram that the rest of this post will focus on:

![[Raw/Media/Resources/8521557063506e8f3b2985e29a03f300_MD5.png]]

**Figure 2: Zoomed in Picture of the Output Network Processing**

With this context, let’s dive into how packet `0427ff35` is being processed.

The purpose of the `raw` table is to provide a way for packets to bypass the connection track (`[conntrack](https://blog.cloudflare.com/conntrack-tales-one-thousand-and-one-flows)`) functionality within iptables. This functionality is not used for this flow, so this table is processed pretty quickly:

PACKET: 2 0427ffe5 OUT=enp3s0 SRC=192.168.1.88 DST=10.43.156.98 LEN=60 \\  
TOS=0x0 TTL=64 ID=45471DF SPORT=49058 DPORT=9898 SYN

TRACE: 2 0427ffe5 raw:OUTPUT:rule:0x18:CONTINUE  -4 -t raw -A OUTPUT \\  
\-p tcp -m tcp --dport 9898 -j TRACE

TRACE: 2 0427ffe5 raw:OUTPUT:return:

TRACE: 2 0427ffe5 raw:OUTPUT:policy:ACCEPT

The first line is informational. It shows the receipt of the packet and the start of the processing. Note that the SRC, `192.168.1.88`, is the source IP address of the node that initiated the curl request.

The second line processed is the `TRACE` that was implemented earlier in this post to produce the captured `xtables-monitor` output (`iptables -t raw -A PREROUTING -p tcp --dport 9898 -J TRACE`).

The third and fourth lines show that the ultimate result of the process on the `raw` table is an `ACCEPT` target. The `ACCEPT` target means to pass the packet to the next table. Recall the relevant tables and chains can be viewed via `iptables` commands.

$ iptables -t raw -L OUTPUT -n --line-numbers  
Chain OUTPUT (policy ACCEPT)  
num  target     prot opt source               destination           
1    cali-OUTPUT  all  --  0.0.0.0/0            0.0.0.0/0            /\* cali:tVnHkvAo15HuiPy0 \*/

$ iptables -t raw -L cali-OUTPUT -n --line-numbers  
Chain cali-OUTPUT (1 references)  
num  target     prot opt source               destination           
1    MARK       all  --  0.0.0.0/0            0.0.0.0/0            /\* cali:njdnLwYeGqBJyMxW \*/ MARK and 0xfff0ffff  
2    cali-to-host-endpoint  all  --  0.0.0.0/0            0.0.0.0/0            /\* cali:rz86uTUcEZAfFsh7 \*/  
3    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            /\* cali:pN0F5zD0b8yf9W1Z \*/ mark match 0x10000/0x10000

$ iptables -t raw -L cali-to-host-endpoint -n --line-numbers  
Chain cali-to-host-endpoint (1 references)  
num  target     prot opt source               destination

The way the above output is traversed is as follows:

![[Raw/Media/Resources/68f021bb92f965e3643d30554c4298ef_MD5.png]]

**Figure 3: Flow Diagram Raw to Mangle**

The first rule in the `cali-OUTPUT` chain is a `MARK` rule. It’s not clear why this rule didn’t show up in the `xtables-monitor` output, but the working theory is that this rule ultimately did not impact how the packet was forwarded and was thus not included. The `MARK` target is a non-terminating target, meaning that processing in the current chain continues even if the `MARK` rule fires.

## State of the Packet

State of the packet leaving the `raw:OUTPUT` chain:

Source IP address: 192.168.1.88  
Source Port: 49058  
Destination IP address: 10.43.156.98  
Destination Port: 9898

As the name suggests, the mangle table is used to modify the packet in some way. A typical modification would be to change the TTL or ToS/DSCP fields inside [the IPv4 header](https://en.wikipedia.org/wiki/Internet_Protocol_version_4#Header). Like `raw:OUTPUT`, the processing here is quick. The packet quickly hits an `ACCEPT` target and processing goes to the next table.

TRACE: 2 0427ffe5 mangle:OUTPUT:return:

TRACE: 2 0427ffe5 mangle:OUTPUT:policy:ACCEPT

## State of the Packet

State of the packet leaving the `mangle:OUTPUT` chain:

Source IP address: 192.168.1.88  
Source Port: 49058  
Destination IP address: 10.43.156.98  
Destination Port: 9898  
Output interface: enp3s0  
Mark: none

There’s a much heavier lift in terms of packet processing in the `nat:OUTPUT` table.

00: PACKET: 2 0427ffe5 OUT=enp3s0 SRC=192.168.1.88 DST=10.43.156.98 LEN=60 \\  
TOS=0x0 TTL=64 ID=45471DF SPORT=49058 DPORT=9898 SYN

01: TRACE: 2 0427ffe5 nat:OUTPUT:rule:0xb9:JUMP:cali-OUTPUT  -4 -t nat \\  
\-A OUTPUT -m comment --comment "cali:tVnHkvAo15HuiPy0" -j cali-OUTPUT

02: TRACE: 2 0427ffe5 nat:cali-OUTPUT:rule:0xb8:JUMP:cali-fip-dnat  -4 \\  
\-t nat -A cali-OUTPUT -m comment --comment "cali:GBTAv2p5CwevEyJm" \\  
\-j cali-fip-dnat

03: TRACE: 2 0427ffe5 nat:cali-fip-dnat:return:

04: TRACE: 2 0427ffe5 nat:cali-OUTPUT:return:

05: TRACE: 2 0427ffe5 nat:OUTPUT:rule:0x9:JUMP:KUBE-SERVICES  -4 \\  
\-t nat -A OUTPUT -m comment --comment "kubernetes service portals" \\  
\-j KUBE-SERVICES

06: TRACE: 2 0427ffe5 nat:KUBE-SERVICES:rule:0x4aaa:JUMP:KUBE-SVC-Y4T5L63IYP3YFEBS  \\  
\-4 -t nat -A KUBE-SERVICES -d 10.43.156.98/32 -p tcp -m comment \\  
\--comment "podinfo/frontend-podinfo:http cluster IP" \\  
\-j KUBE-SVC-Y4T5L63IYP3YFEBS

07: TRACE: 2 0427ffe5 nat:KUBE-SVC-Y4T5L63IYP3YFEBS:rule:0x471b:JUMP:KUBE-MARK-MASQ \\  
\-4 -t nat -A KUBE-SVC-Y4T5L63IYP3YFEBS ! -s 10.42.0.0/16 -d 10.43.156.98/32 \\  
\-p tcp -m comment --comment "podinfo/frontend-podinfo:http cluster IP" \\  
\-j KUBE-MARK-MASQ

08: TRACE: 2 0427ffe5 nat:KUBE-MARK-MASQ:rule:0x4aa4:CONTINUE  \\  
\-4 -t nat -A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000

09: TRACE: 2 0427ffe5 nat:KUBE-MARK-MASQ:return:

10: TRACE: 2 0427ffe5 nat:KUBE-SVC-Y4T5L63IYP3YFEBS:rule:0x471d:JUMP:KUBE-SEP-OYNVDBAPYJ623W6H  \\  
\-4 -t nat -A KUBE-SVC-Y4T5L63IYP3YFEBS -m comment \\  
\--comment "podinfo/frontend-podinfo:http -> 10.42.54.194:9898" \\  
\-j KUBE-SEP-OYNVDBAPYJ623W6H

11: TRACE: 2 0427ffe5 nat:KUBE-SEP-OYNVDBAPYJ623W6H:rule:0x4721:ACCEPT  \\  
\-4 -t nat -A KUBE-SEP-OYNVDBAPYJ623W6H -p tcp -m comment \\  
\--comment "podinfo/frontend-podinfo:http" -m tcp \\  
\-j DNAT --to-destination 10.42.54.194:9898

The first five lines show processing going from `nat:OUTPUT`, to `nat:cali-OUTPUT`, to `nat:cali-fip-dnat`, before returning all the way to `nat:OUTPUT` and processing the next rule in that chain which is `nat:KUBE-SERVICES`.

Processing in `nat:KUBE-SERVICES` chain starts in the fifth line and the next matching chain is `nat:KUBE-SVC-Y4T5L63IYP3YFEBS` based on the destination IP address of the service (10.43.156.98).

Line eight then forwards processing to the `nat:KUBE-MARK-MASQ` chain since the packet's source address is not in the 10.42.0.0/16 subnet (basically, the source isn’t another pod). The sole entry in this chain marks the packet with `0x4000`, done in line nine. This mark has node local significance and does not follow the packet to any other server/hop once leaving this node. It will be used later in the processing of the packet. Line 10 then returns processing to the `nat:KUBE-SVC-Y4T5L63IYP3YFEBS` chain.

Line 11 is where the load balancing within kube-proxy happens in iptables mode, which is best seen in the iptables output for the chain:

$ iptables -t nat -L KUBE-SVC-Y4T5L63IYP3YFEBS -n --line-numbers  
Chain KUBE-SVC-Y4T5L63IYP3YFEBS (1 references)  
num  target     prot opt source               destination           
1    KUBE-MARK-MASQ  tcp  -- !10.42.0.0/16         10.43.156.98           
/\* podinfo/frontend-podinfo:http cluster IP \*/

2    KUBE-SEP-6G2GMHWEBXQ5W3DV  all  --  0.0.0.0/0            0.0.0.0/0       
/\* podinfo/frontend-podinfo:http -> 10.42.217.66:9898 \*/   
statistic mode random probability 0.50000000000

3    KUBE-SEP-OYNVDBAPYJ623W6H  all  --  0.0.0.0/0            0.0.0.0/0              
/\* podinfo/frontend-podinfo:http -> 10.42.54.194:9898 \*/

Note the presence of `statistic mode random probability 0.50000000000` in the second rule, which means exactly what it sounds like. This rule will fire 50% of the time and forward processing of the packet to the `nat:KUBE-SEP-6G2GMHWEBXQ5W3DV` chain. When this chain is not selected, processing falls through to the `nat:KUBE-SEP-OYNVDBAPYJ623W6H` chain. These two `SEP` chains each have the same function: a destination NAT action to update the destination IP address from the service address of `10.43.156.98`to that of one of the two pods deployed for the Podinfo service. The chain ending in `W3DV` forwards to the pod with IP address `10.42.217.66`, and the chain ending in `3W6h` forwards to the pod with IP address `10.43.54.194`. This particular packet happened to load balance `10.43.54.194` so line 11 shows the jump from the `SVC` chain to the `SEP` chain for the `10.43.54.194` pod. Finally, line 12 show the `DNAT` action actually happening and the destination IP address is now `10.42.54.194`.

![[Raw/Media/Resources/e11b7f6ee0f17e05512b965cce9d7fe5_MD5.png]]

**Figure 4: Flow Diagram NAT to Filter**

## State of the Packet

State of the packet leaving the `nat:OUTPUT` chain:

Source IP address: 192.168.1.88  
Source Port: 49058  
Destination IP address: 10.42.54.194  
Destination Port: 9898  
Output interface: enp3s0  
Mark: 0x4000/0x4000

The filter table is used to restrict traffic from leaving the host. With that description, no significant change in processing is anticipated here.

00: PACKET: 2 0427ffe5 OUT=enp3s0 SRC=192.168.1.88 DST=10.42.54.194 \\  
LEN=60 TOS=0x0 TTL=64 ID=45471DF SPORT=49058 DPORT=9898 SYN MARK=0x4000

01: TRACE: 2 0427ffe5 filter:OUTPUT:rule:0x91:JUMP:cali-OUTPUT  -4 \\  
\-t filter -A OUTPUT -m comment --comment "cali:tVnHkvAo15HuiPy0" \\  
\-j cali-OUTPUT

02: TRACE: 2 0427ffe5 filter:cali-OUTPUT:rule:0x8a:CONTINUE  -4 \\  
\-t filter -A cali-OUTPUT -m comment --comment "cali:iC1pSPgbvgQzkUk\_" \\  
\-j MARK --set-xmark 0x0/0xf0000

03: TRACE: 2 0427ffe5 filter:cali-OUTPUT:return:

04: TRACE: 2 0427ffe5 filter:OUTPUT:rule:0x17:JUMP:KUBE-PROXY-FIREWALL  -4 \\  
\-t filter -A OUTPUT -m conntrack --ctstate NEW -m comment \\  
\--comment "kubernetes load balancer firewall" -j KUBE-PROXY-FIREWALL

05: TRACE: 2 0427ffe5 filter:KUBE-PROXY-FIREWALL:return:

06: TRACE: 2 0427ffe5 filter:OUTPUT:rule:0x12:JUMP:KUBE-SERVICES  -4 \\  
\-t filter -A OUTPUT -m conntrack --ctstate NEW -m comment \\  
\--comment "kubernetes service portals" -j KUBE-SERVICES

07: TRACE: 2 0427ffe5 filter:KUBE-SERVICES:return:

08: TRACE: 2 0427ffe5 filter:OUTPUT:rule:0x6:JUMP:KUBE-FIREWALL  -4 \\  
\-t filter -A OUTPUT -j KUBE-FIREWALL

09: TRACE: 2 0427ffe5 filter:KUBE-FIREWALL:return:

10: TRACE: 2 0427ffe5 filter:OUTPUT:return:

11: TRACE: 2 0427ffe5 filter:OUTPUT:policy:ACCEPT

The first `TRACE` entry shows the processing of the packet from the `filter:OUTPUT` chain to the `filter:cali-OUTPUT` chain. The `filter:cali-OUTPUT` chain updates the marking on the packet and then passes processing back to the `filter:OUTPUT` chain in line three (note, the marking on the packet doesn’t actually change due to the bitwise `AND` operation between `0xf000`and `0x4000`). The rest of the processing goes through various Kubernetes specific chains, which ultimately make no changes and accept the packet.

This execution does perform some useful processing, however. Note the lines that have `-m conntrack -ctstate NEW` in them. Since this is a TCP SYN packet (the first packet in a three-way handshake), this packet is subject to more scrutiny. The aforementioned lines serve to update the connection table used by iptables. Note that in the full raw `xtable-monitor --trace` [output](https://raw.githubusercontent.com/TheFutonEng/k8s-svc-trace/main/trace.txt), subsequent packets have far fewer lines.

## State of the Packet

State of the packet leaving the `nat:OUTPUT` chain:

Source IP address: 192.168.1.88  
Source Port: 49058  
Destination IP address: 10.42.54.194  
Destination Port: 9898  
Output interface: enp3s0  
Mark: 0x4000/0x4000

Again, the purpose of the mangle tables is to modify the packet, mostly in the TTL and ToS/DSCP headers. Given that, no major changes are expected here either.

00: PACKET: 2 0427ffe5 OUT=vxlan.calico SRC=192.168.1.88 DST=10.42.54.194 \\  
LEN=60 TOS=0x0 TTL=64 ID=45471DF SPORT=49058 DPORT=9898 SYN MARK=0x4000

01: TRACE: 2 0427ffe5 mangle:POSTROUTING:rule:0x15:JUMP:cali-POSTROUTING  -4 \\  
\-t mangle -A POSTROUTING -m comment --comment "cali:O3lYWMrLQYEMJtB5" \\  
\-j cali-POSTROUTING

02: TRACE: 2 0427ffe5 mangle:cali-POSTROUTING:rule:0x12:CONTINUE  -4 \\  
\-t mangle -A cali-POSTROUTING -m comment --comment "cali:nnqPh8lh2VOogSzX" \\  
\-j MARK --set-xmark 0x0/0xf0000

03: TRACE: 2 0427ffe5 mangle:cali-POSTROUTING:rule:0x13:JUMP:cali-to-host-endpoint  \\  
\-4 -t mangle -A cali-POSTROUTING -m comment --comment "cali:nquN8Jw8Tz72pcBW" \\  
\-m conntrack --ctstate DNAT -j cali-to-host-endpoint

04: TRACE: 2 0427ffe5 mangle:cali-to-host-endpoint:return:

05: TRACE: 2 0427ffe5 mangle:cali-POSTROUTING:return:

06: TRACE: 2 0427ffe5 mangle:POSTROUTING:return:

07: TRACE: 2 0427ffe5 mangle:POSTROUTING:policy:ACCEPT

Another “updated marking” happens in line 2 but the operation doesn’t change the mark from `0x40000`.

## State of the Packet

State of the packet leaving the `nat:OUTPUT` chain:

Source IP address: 192.168.1.88  
Source Port: 49058  
Destination IP address: 10.42.54.194  
Destination Port: 9898  
Output interface: vxlan.calico  
Mark: 0x4000/0x4000

A more notable change in the state of the packet is indirectly due to an iptables modification. The destination IP address of the packet changed in the `nat:OUTPUT`chain from the service address of`10.43.156.98` to the pod address of `10.42.54.194`. A routing lookup against the pod address happened after this change, and the outgoing interface is indeed `vxlan.calico`:

$ ip route get 10.42.54.194  
10.42.54.194 via 10.42.54.193 dev vxlan.calico src 10.42.135.130 uid 0   
    cache 

This means that the VxLAN overlay will deliver the packet from the node where the request was initiated to the node where the pod is running, and the “state of the packet” headers that have been tracking in this post will be what the destination node processes after popping off the VxLAN encapsulation. All of the VxLAN processing happens outside of iptables.

![[Raw/Media/Resources/8c69e22ba81d887df231e8adf2e35173_MD5.gif]]

**Figure 5: I told you this was going to be brain hurty…**

The NAT table would handle any source or destination NAT changes to the packet. Since the destination was already updated, no further modifications really happened here:

00: PACKET: 2 0427ffe5 OUT=vxlan.calico SRC=192.168.1.88 DST=10.42.54.194 LEN=60 TOS=0x0 TTL=64 ID=45471DF SPORT=49058 DPORT=9898 SYN MARK=0x4000

01: TRACE: 2 0427ffe5 nat:POSTROUTING:rule:0xba:JUMP:cali-POSTROUTING  -4 \\  
\-t nat -A POSTROUTING -m comment --comment "cali:O3lYWMrLQYEMJtB5" \\  
\-j cali-POSTROUTING

02: TRACE: 2 0427ffe5 nat:cali-POSTROUTING:rule:0xb5:JUMP:cali-fip-snat  -4 \\  
\-t nat -A cali-POSTROUTING -m comment --comment "cali:Z-c7XtVd2Bq7s\_hA" \\  
\-j cali-fip-snat

03: TRACE: 2 0427ffe5 nat:cali-fip-snat:return:

04: TRACE: 2 0427ffe5 nat:cali-POSTROUTING:rule:0xb6:JUMP:cali-nat-outgoing  \\  
\-4 -t nat -A cali-POSTROUTING -m comment --comment "cali:nYKhEzDlr11Jccal" \\  
\-j cali-nat-outgoing

05: TRACE: 2 0427ffe5 nat:cali-nat-outgoing:return:

06: TRACE: 2 0427ffe5 nat:cali-POSTROUTING:rule:0xb7:ACCEPT  -4 -t nat \\  
\-A cali-POSTROUTING -o vxlan.calico -m comment \\  
\--comment "cali:e9dnSgSVNmIcpVhP" -m addrtype ! --src-type LOCAL \\  
\--limit-iface-out -m addrtype --src-type LOCAL -j MASQUERADE --random-fully

If you made it this far, I’m honestly shocked. Thanks for reading, and I hope you got something out of this article besides a headache and a disdain for iptables.