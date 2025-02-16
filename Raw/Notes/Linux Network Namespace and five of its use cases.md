---
title: Linux Network Namespace and five of its use cases
source: https://ramesh-sahoo.medium.com/linux-network-namespace-and-five-use-cases-using-various-methods-f45b1ec5db8f
clipped: 2023-09-04
published: 
category: development
tags:
  - network
  - low-level
read: false
---

Linux network namespace

> The [Network namespace](https://man7.org/linux/man-pages/man7/network_namespaces.7.html) isolates the system’s **physical** network from the **virtual** network **namespace** within a single system. Each network namespace has its own interfaces, routing tables, forwarding rules, etc. Processes can be launched and dedicated to one network namespace. Linux network network namespace is widely used in OpenStack, docker, podman, cri-o, more..

A physical network device can live in exactly one network  
namespace. When a network namespace is freed (i.e., when the  
last process in the namespace terminates), its physical network  
devices are moved back to the initial network namespace (not to  
the parent of the process).

A virtual network (`**veth**`) device pair provides a pipe-like  
abstraction that can be used to create tunnels between network  
namespaces, and can be used to create a bridge to a physical  
network device in another namespace. When a namespace is freed,  
the `**veth**` devices that it contains are destroyed.

Use of network namespaces requires a kernel that is configured  
with the `**CONFIG_NET_NS**` option.

> In this article, we’ll use various use cases of Linux network namespace with precise example using Linux native tool and `**OpenVswitch**`.

## Network namespace use case — 1

![[Raw/Notes/Raw/Media/Resources/a2526b3824b2c4874fdf9f08d00549b6_MD5.png]]

> Create two network namespaces `**red**` and `**green**`**.** Then **c**reate two different veth pairs `**eth0-r** → **veth-r**` and `**eth0-g** → **veth-g**`**.** Move `**eth0-r**` to the red namespace and `**eth0-g**` to the green namespace and then attach the other end of veth pairs `**veth-r**` and `**veth-g**` to the O**penVswitch** bridge `**OVS1**`. Assign the IP `**10.0.0.1/24**` to the `**eth0-r**` in the `red` namespace and `**10.0.0.2/2**4` to the `**eth0-g**` in the green namespace. In the end communicate with each other network namespace IP using `**ping**` command.

**Prerequisites for this this LAB**  
\- **Linux system** → For this lab, I used a **Ubuntu** system but any Linux distribution can be used to complete this exercise.  
\- Installation of **OpenVswitch**

1.System’s present root network namespace looks as following, here we have a local interface(`**lo**`) and a physical network interface(`**enp0s1**`**)**.

root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc fq\_codel state UP mode DEFAULT group default qlen 1000  
    link/ether c2:f6:aa:e3:7d:f5 brd ff:ff:ff:ff:ff:f

2. Now lets create the `**red**` and `**green**` network namespace and look for their existence through various methods.

  
root@linux:~  
root@linux:~

  
root@linux:~  
green  
Red

  
root@ubuntu:~  
total 0  
\-r--r--r-- 1 root root 0 May  2 10:48 green  
\-r--r--r-- 1 root root 0 May  2 10:47 red

  
root@linux:~

In the following output, inside the `**red**` and `**green**` namespace there is only presence of local interface(`**lo**`), this is because no other **network** **interface** has been added to the specific namespace.

root@linux:~  
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

root@linux:~  
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

3. Now lets create a virtual switch/bridge `**OVS1**` using the `**ovs-vsctl**`(OpenVswitch) command. This switch will provide connectivity for the `**red**` and `**green**` namespaces to communicate with each other.

  
root@linux:~

  
root@linux:~  
OVS1

  
root@linux:~  
f7041f1f-2878\-4721\-8bd7-350fc731c4c5  
    Bridge OVS1  
        Port OVS1  
            Interface OVS1  
                type: internal  
    ovs\_version: "2.17.3"

Now in the `**root**` network namespace we have `**OVS1**` switch network interface but the link status is still `**down**` because no device is connected to the switch port. The interface `**ovs-system**` is the default interface created by the **OpenVswitch** when creating an **OVS** bridge/switch.

root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc fq\_codel state UP mode DEFAULT group default qlen 1000  
    link/ether c2:f6:aa:e3:7d:f5 brd ff:ff:ff:ff:ff:ff  
3: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 26:b6:fb:41:6f:e9 brd ff:ff:ff:ff:ff:ff  
4: OVS1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 0a:64:6d:54:00:41 brd ff:ff:ff:ff:ff:ff

To connect the `**red**` and `**green**` network namespace to the switch `**OVS1**` switch, we’ll use `**VETH**`(Virtual network interfaces) interface.

## What is **VETH**(Virtual network interfaces) interface?

The `**VETH**` devices are virtual Ethernet devices and created in pairs. Packets transmitted on one device in the pair are immediately received on the other device. When either device is down, the link state of the pair is down.

4. Create two `**VETH**` interface, one for the `**red**` and another for the `**green**` network namespace.

  
root@linux:~

  
root@linux:~

Now the VETH pairs (`**eth0-r** & **veth-r**`) and (`**eth0-g** & **veth-g**`) are showing in the `**root**` namespace.

root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc fq\_codel state UP mode DEFAULT group default qlen 1000  
    link/ether c2:f6:aa:e3:7d:f5 brd ff:ff:ff:ff:ff:ff  
3: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 26:b6:fb:41:6f:e9 brd ff:ff:ff:ff:ff:ff  
4: OVS1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 0a:64:6d:54:00:41 brd ff:ff:ff:ff:ff:ff  
5: veth-r@eth0-r: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether ea:1d:fc:9a:1e:ce brd ff:ff:ff:ff:ff:ff  
6: eth0\-r@veth-r: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 4a:be:26:7f:31:7c brd ff:ff:ff:ff:ff:ff  
7: veth-g@eth0-g: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 66:27:a3:bc:58:4e brd ff:ff:ff:ff:ff:ff  
8: eth0\-g@veth-g: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 5e:51:37:dd:f4:da brd ff:ff:ff:ff:ff:ff 

In the above output the naming convention of the **veth** pairs `**veth-r@eth0-r → eth0-r@veth-r**` and `**veth-g@eth0-g → eth0-g@veth-g**` confirms that these are **veth** interfaces.

5. Move one end of the `**VETH**` interface to their designated network namespace and connect the other end to the `**OVS1**` bridge/switch. Here we’ll move the pair `**eth0-r**` and `**eth0-g**` to the **red** and **green** namespaces and then attach the other veth pair `**veth-r**` and `**veth-g**` to the `**OVS1**` bridge/switch.

  
root@linux:~

  
root@linux:~

Now from the root network namespace the `**eth0-r**` and `**eth0-g**` veth pairs have disappeared and we can only see the other pairs `**veth-r**` and `**veth-g**`. This is because the `**eth0-r**` and `**eth0-g**` veth interfaces were moved to their isolated network namespace `**red**` and `**green**`.

root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc fq\_codel state UP mode DEFAULT group default qlen 1000  
    link/ether c2:f6:aa:e3:7d:f5 brd ff:ff:ff:ff:ff:ff  
3: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 26:b6:fb:41:6f:e9 brd ff:ff:ff:ff:ff:ff  
4: OVS1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 0a:64:6d:54:00:41 brd ff:ff:ff:ff:ff:ff  
5: veth-r@if6: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether ea:1d:fc:9a:1e:ce brd ff:ff:ff:ff:ff:ff link\-netns red  
6: veth-g@if8: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 66:27:a3:bc:58:4e brd ff:ff:ff:ff:ff:ff link\-netns green

The above output confirms that `**veth-r@if6**` interface is part of `**red**` namespace and the `**veth-g@if8**` is part of the `**green**` namespace.

In the following output, we can see that the `**eth0-r**` and `**eth0-g**` veth interfaces are attached to the `**red**` and `**green**` network namespace but their status is **down**, this is because link of these **veth** pairs are not **UP** yet.

root@linux:~  
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
6: eth0\-r@if5: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 4a:be:26:7f:31:7c brd ff:ff:ff:ff:ff:ff link\-netnsid 0

root@linux:~  
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
8: eth0\-g@if7: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 5e:51:37:dd:f4:da brd ff:ff:ff:ff:ff:ff link\-netnsid 0

6. Now let’s connect the other end `**veth-r**` and `**veth-g**` to the `**OVS1**` switch/bridge using the command `ovs-vsctl`.

  
root@linux:~

  
root@linux:~

  
root@linux:~  
f7041f1f-2878\-4721\-8bd7-350fc731c4c5  
    Bridge OVS1  
        Port OVS1  
            Interface OVS1  
                type: internal  
        Port veth-r  
            Interface veth-r  
        Port veth-g  
            Interface veth-g  
    ovs\_version: "2.17.3"

  
root@linux:~  
veth-r  
veth-g

  
root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc fq\_codel state UP mode DEFAULT group default qlen 1000  
    link/ether c2:f6:aa:e3:7d:f5 brd ff:ff:ff:ff:ff:ff  
3: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 26:b6:fb:41:6f:e9 brd ff:ff:ff:ff:ff:ff  
4: OVS1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 0a:64:6d:54:00:41 brd ff:ff:ff:ff:ff:ff  
5: veth-r@if6: <BROADCAST,MULTICAST> mtu 1500 qdisc noop master ovs-system state DOWN mode DEFAULT group default qlen 1000  
    link/ether ea:1d:fc:9a:1e:ce brd ff:ff:ff:ff:ff:ff link\-netns red  
7: veth-g@if8: <BROADCAST,MULTICAST> mtu 1500 qdisc noop master ovs-system state DOWN mode DEFAULT group default qlen 1000  
    link/ether 66:27:a3:bc:58:4e brd ff:ff:ff:ff:ff:ff link\-netns green

Now the two network interfaces `**red**` and `**green**` have path to each other via **OpenVswitch** `**OVS1**`.

![[Raw/Notes/Raw/Media/Resources/e5c49bb14138cd5b55ae23430eb259ee_MD5.png]]

7. Now as a next step we will configure the `**eth0-r**` and `**eth0-g**` **veth** interfaces inside the `**red**` and `**green**` namespace. Here we will bring up necessary interfaces within and outside namespace after assigning an IP address to the `**eth0-r**` and `**eth0-g**` interfaces inside the `**red**` and `**green**` namespace.

Bring up the `**eth0-r**` VETH interface inside the `**red**` namespace and assign the IP `**10.0.0.1/24**` to this interface.

root@linux:~  
root@linux:~

  
root@linux:~

  
root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
6: eth0\-r@if5: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN mode DEFAULT group default qlen 1000  
    link/ether 4a:be:26:7f:31:7c brd ff:ff:ff:ff:ff:ff link\-netnsid 0

  
root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
    inet 127.0.0.1/8 scope host lo  
       valid\_lft forever preferred\_lft forever  
    inet6 ::1/128 scope host   
       valid\_lft forever preferred\_lft forever  
6: eth0\-r@if5: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN group default qlen 1000  
    link/ether 4a:be:26:7f:31:7c brd ff:ff:ff:ff:ff:ff link\-netnsid 0  
    inet 10.0.0.1/24 scope global eth0\-r  
       valid\_lft forever preferred\_lft forever

  
root@linux:~  
10.0.0.0/24 dev eth0\-r proto kernel scope link src 10.0.0.1 linkdown 

In the above output the link status for the `**eth0-r**` inside the `**red**` namespace shows as `**state LOWERLAYERDOWN**`. This is because the ethernet link of other end `**veth-r**`(connected to `**OVS1**` bridge) of this veth pair is not yet **up** and we have to manually bring that interface up.

Here the `**root**` network namespace doesn’t have any awareness about the route for `**10.0.0.0/24**`. This is because `**10.0.0.0/24**` is part of the isolated network namespace `**red**`.

root@linux:~  
default via 192.168.205.1 dev enp0s1 proto dhcp src 192.168.205.8 metric 100   
192.168.205.0/24 dev enp0s1 proto kernel scope link src 192.168.205.8 metric 100

Bring up the `**eth0-g**` VETH interface inside the `**green**` namespace and assign the IP `**10.0.0.2/24**` to this interface.

  
root@linux:~  
root@linux:~  
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
8: eth0\-g@if7: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 5e:51:37:dd:f4:da brd ff:ff:ff:ff:ff:ff link\-netnsid 0

  
root@linux:~  
root@linux:~  
root@linux:~

  
root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
    inet 127.0.0.1/8 scope host lo  
       valid\_lft forever preferred\_lft forever  
    inet6 ::1/128 scope host   
       valid\_lft forever preferred\_lft forever  
8: eth0\-g@if7: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN group default qlen 1000  
    link/ether 5e:51:37:dd:f4:da brd ff:ff:ff:ff:ff:ff link\-netnsid 0  
    inet 10.0.0.2/24 scope global eth0\-g  
       valid\_lft forever preferred\_lft forever

  
root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
8: eth0\-g@if7: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN mode DEFAULT group default qlen 1000  
    link/ether 5e:51:37:dd:f4:da brd ff:ff:ff:ff:ff:ff link\-netnsid 0

  
root@linux:~  
10.0.0.0/24 dev eth0\-g proto kernel scope link src 10.0.0.2 linkdown

8. Now we will bring up the `**veth-r**` and `**veth-g**` interfaces in the root network namespace which will bring up the entire link.

  
root@linux:~  
root@linux:~

  
root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc fq\_codel state UP mode DEFAULT group default qlen 1000  
    link/ether c2:f6:aa:e3:7d:f5 brd ff:ff:ff:ff:ff:ff  
3: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 26:b6:fb:41:6f:e9 brd ff:ff:ff:ff:ff:ff  
4: OVS1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 0a:64:6d:54:00:41 brd ff:ff:ff:ff:ff:ff  
5: veth-r@if6: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue master ovs-system state UP mode DEFAULT group default qlen 1000  
    link/ether ea:1d:fc:9a:1e:ce brd ff:ff:ff:ff:ff:ff link\-netns red  
7: veth-g@if8: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue master ovs-system state UP mode DEFAULT group default qlen 1000  
    link/ether 66:27:a3:bc:58:4e brd ff:ff:ff:ff:ff:ff link\-netns green

  
root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
6: eth0\-r@if5: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether 4a:be:26:7f:31:7c brd ff:ff:ff:ff:ff:ff link\-netnsid 0

  
root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
8: eth0\-g@if7: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether 5e:51:37:dd:f4:da brd ff:ff:ff:ff:ff:ff link\-netnsid 0

After bringing up the network interface `**veth-r**` and `**veth-g**` inside the `**root**` namespace, the above output shows that the link status as `**UP**` for the `**eth0-r**` and `**eth0-g**` network interface ( `**<UP,LOWER_UP>**` ) and the link status for the `**veth-r**` and `**veth-g**` shows as `**<UP,LOWER_UP>**`, `**master ovs-system state UP**`.

Now this is the state of the network diagram where `**red**` and `**green**` namespaces are connected with each other through the `**OVS1**` switch.

![[Raw/Notes/Raw/Media/Resources/a2526b3824b2c4874fdf9f08d00549b6_MD5.png]]

9. In the following output, the `**ping**` command successfully confirms that connectivity between `**red**` and `**green**` namespace and this the end of the `**network namespace use case 1**`.

  
root@linux:~  
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.  
64 bytes from 10.0.0.2: icmp\_seq=1 ttl=64 time\=1.35 ms  
64 bytes from 10.0.0.2: icmp\_seq=2 ttl=64 time\=0.267 ms  
\--- 10.0.0.2 ping statistics ---  
2 packets transmitted, 2 received, 0% packet loss, time 1006ms  
rtt min/avg/max/mdev = 0.267/0.806/1.345/0.539 ms

  
root@linux:~  
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.  
64 bytes from 10.0.0.1: icmp\_seq=1 ttl=64 time\=1.66 ms  
64 bytes from 10.0.0.1: icmp\_seq=2 ttl=64 time\=0.230 ms  
\--- 10.0.0.1 ping statistics ---  
2 packets transmitted, 2 received, 0% packet loss, time 1005ms  
rtt min/avg/max/mdev = 0.230/0.944/1.659/0.714 ms

## Network namespace use case — 2

![[Raw/Notes/Raw/Media/Resources/e2dc2bbf1a5d20f7385bfc8d59734923_MD5.png]]

> In this exercise we’ll use the Linux native tool(without **OpenVswitch**) to create two network namespaces `**net1**` and `**net2**` then create a **veth** pair `**veth1**` and `**veth2**`. Here the `**veth1**` network interface from the `**net1**` network namespace is connected to the `**veth2**` network interface in the `**net2**` network namespace like an ethernet **cross cable** between two physical systems.

**Prerequisites for this this LAB  
**— **Linux system** → For this lab, I used a **Ubuntu** system but any Linux distribution can be used to complete this exercise.

1.Create the isolated network namespaces `**net1**` and `**net2**`.

  
root@linux:~  
root@linux:~

  
root@linux:~  
net2  
net1

2. Create a veth pair `**veth1**` and `**veth2**`, connect the `**veth1**` to the `**net1**` namespace and the other end `**veth2**` to `**net2**` namespace.

  
root@linux:~

  
root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc fq\_codel state UP mode DEFAULT group default qlen 1000  
    link/ether c2:f6:aa:e3:7d:f5 brd ff:ff:ff:ff:ff:ff

  
root@linux:~  
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
2: veth1@if2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 8e:f2:4f:3f:a2:a6 brd ff:ff:ff:ff:ff:ff link\-netns net2

  
root@linux:~  
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
2: veth2@if2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000  
    link/ether 06:0c:de:d5:81:b4 brd ff:ff:ff:ff:ff:ff link\-netns net1

3. Configure the `**veth1**` interface in the `**net1**` namespace and assign the IP `**192.168.20.1/24**` to this interface.

  
root@linux:~

  
root@linux:~

  
root@linux:~  
root@linux:~

  
root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
2: veth1@if2: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default qlen 1000  
    link/ether 8e:f2:4f:3f:a2:a6 brd ff:ff:ff:ff:ff:ff link\-netns net2

  
root@linux:~  
192.168.20.0/24 dev veth1 proto kernel scope link src 192.168.20.1 linkdown 

root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
    inet 127.0.0.1/8 scope host lo  
       valid\_lft forever preferred\_lft forever  
    inet6 ::1/128 scope host   
       valid\_lft forever preferred\_lft forever  
2: veth1@if2: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000  
    link/ether 8e:f2:4f:3f:a2:a6 brd ff:ff:ff:ff:ff:ff link\-netns net2  
    inet 192.168.20.1/24 scope global veth1  
       valid\_lft forever preferred\_lft forever

4. Configure the `**veth2**` interface in the `**net2**` namespace and assign the IP `**192.168.20.2/24**` to this interface.

  
root@linux:~

  
root@linux:~  
root@linux:~  
root@linux:~

  
root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
2: veth2@if2: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether 06:0c:de:d5:81:b4 brd ff:ff:ff:ff:ff:ff link\-netns net1

  
root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
    inet 127.0.0.1/8 scope host lo  
       valid\_lft forever preferred\_lft forever  
    inet6 ::1/128 scope host   
       valid\_lft forever preferred\_lft forever  
2: veth2@if2: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000  
    link/ether 06:0c:de:d5:81:b4 brd ff:ff:ff:ff:ff:ff link\-netns net1  
    inet 192.168.20.2/24 scope global veth2  
       valid\_lft forever preferred\_lft forever  
    inet6 fe80::40c:deff:fed5:81b4/64 scope link   
       valid\_lft forever preferred\_lft forever  
root@linux:~  
192.168.20.0/24 dev veth2 proto kernel scope link src 192.168.20.2

  
root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
2: veth1@if2: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether 8e:f2:4f:3f:a2:a6 brd ff:ff:ff:ff:ff:ff link\-netns net2

The `**root**` namespace still doesn’t have any awareness about the IP configuration of the `**net1**` and `**net2**` namespace.

root@linux:~  
exit  
root@linux:~  
exit

root@linux:~  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc fq\_codel state UP mode DEFAULT group default qlen 1000  
    link/ether c2:f6:aa:e3:7d:f5 brd ff:ff:ff:ff:ff:ff

root@linux:~  
default via 192.168.205.1 dev enp0s1 proto dhcp src 192.168.205.8 metric 100   
192.168.205.0/24 dev enp0s1 proto kernel scope link src 192.168.205.8 metric 100   
192.168.205.1 dev enp0s1 proto dhcp scope link src 192.168.205.8 metric 100

Ping test to the `**veth1**` IP fails from the `**root**` network namespace. This is because the IP `**192.168.20.1**` belong to the isolated network namespace `**net1**`.

root@linux:~  
PING 192.168.20.1 (192.168.20.1) 56(84) bytes of data.

\--- 192.168.20.1 ping statistics ---  
1 packets transmitted, 0 received, 100% packet loss, time 0ms

5. Now test network communication from the `**net1**` and `**net2**` namespace using the `**ping**` command. The following output confirms that `**veth1**` interface in the `**net1**` namespace is able to communicate to the `**veth2**` interface in the `**net2**` namespace successfully.

root@linux:~  
PING 192.168.20.2 (192.168.20.2) 56(84) bytes of data.  
64 bytes from 192.168.20.2: icmp\_seq=1 ttl=64 time\=0.169 ms

\--- 192.168.20.2 ping statistics ---  
1 packets transmitted, 1 received, 0% packet loss, time 0ms  
rtt min/avg/max/mdev = 0.169/0.169/0.169/0.000 ms

root@linux:~  
PING 192.168.20.1 (192.168.20.1) 56(84) bytes of data.  
64 bytes from 192.168.20.1: icmp\_seq=1 ttl=64 time\=0.106 ms

\--- 192.168.20.1 ping statistics ---  
1 packets transmitted, 1 received, 0% packet loss, time 0ms  
rtt min/avg/max/mdev = 0.106/0.106/0.106/0.000 ms

## Network namespace use case — 3

![[Raw/Notes/Raw/Media/Resources/7ce105bc3db4cb7f09bcde7238846d7a_MD5.png]]

> In this exercise the the network namespaces `**net1**` and `**net2**` are connected with each other via `**OVS1**` OpenVswitch with **VLAN** tag 10. For this exercise, create two network namespace `**net1**` and `**net2**`. Create two veth pairs `**eth0-n1 → veth-n1**` and `**eth0-n2 → veth-n2**`. Move the `**eth0-n1**` to the `**net1**` network namespace and move the `**eth0-n2**` to the `**net2**` network namespace. Attach the other ends `**veth-n1**` and `**veth-n2**` to the `**OVS1**` OpenVswitch and assign **VLAN** tag **10** to the **OVS** ports. Then configure the IP address `192.168.21.1/24` to the **eth0-n1** and `192.168.21.2` to the **eth0-n2** network interface in their respective namespace.

**Prerequisites for this this LAB**  
— **Linux system** → For this lab, I used a **Ubuntu** system but any Linux distribution can be used to complete this exercise.  
\- Installation of **OpenVswitch**

1.Create the `net1` and `**net2**` network namespace and then create the **veth** pairs `***eth0-n1 → veth-n1***` and `***eth0-n2 → veth-n2.***`

\# ip netns add net1  
\# ip netns add net2  
\# ip link add eth0-n1 type veth peer name veth-n1  
\# ip link add eth0-n2 type veth peer name veth-n2

2. Move the `**eth0-n1**` network interface to `**net1**` and `**eth0-n2**` network interface to the `**net2**` network namespace.

\# ip link set eth0-n1 netns net1  
\# ip link set eth0-n2 netns net2

3. Attach other ends of the **veth** pair `**veth-n1**` and `**veth-n2**` to `**OVS1**` switch then set `**vlan**` tag 10 to each port. Refer to the **exercise 1** for the steps to create `**OVS1**` bridge using `**ovs-vsctl**` command.

\# ovs-vsctl add-port OVS1 veth-n1  
\# ovs-vsctl add-port OVS1 veth-n2  
\# ovs-vsctl set port veth-n1 tag=10   
\# ovs-vsctl set port veth-n2 tag=10   
\# ovs-vsctl show

To remove vlan tag use the following command   
\# ovs-vsctl remove port veth-n1 tag 10 

4. Bring bring up the `**veth-n1**` and `**veth-n2**` network interfaces inside the `**root**` network namespace.

\# ip link set dev veth-n1 up  
\# ip link set dev veth-n2 up

5. Configure IP address `**192.168.21.1/24**` to the `**eth0-n1**` veth interface in the `**net1**` namespace and `**192.168.21.2/24**` IP address to the `**eth0-n2**` veth interfaces in the `**net2**` namespace.

\# Command execution in the net1 namespace  
\# ip netns exec net1 bash  
\# ip link set dev lo up  
\# ip link set dev eth0-n1 up  
\# ip link   
\# ip address add 192.168.21.1/24 dev eth0-n1  
\# ip r   
\# Command execution in the net2 namespace  
\# ip netns exec net2 bash  
\# ip link set dev lo up  
\# ip link set dev eth0-n2 up  
\# ip link   
\# ip address add 192.168.21.2/24 dev eth0-n2  
\# ip r 

6. Now test network connectivity from each other namespace using the `**ping**` command and make sure that `**ping**` command gets successful ICMP response.

## Network namespace use case — 4

![[Raw/Notes/Raw/Media/Resources/f1dff44f3f5ca844174bd29db1404758_MD5.png]]

> In this exercise:

-   `**Red**` and **green** namespaces are separated by **VLAN** tag 100 and 200 so that they can’t communicate/conflict with each other even though they share the same subnet and IP address.
-   Two **DHCP** servers provided by `**dnsmasq**` run separately in the `**DHCP-R**` and `**DHCP-G**` namespace.
-   **TAP** interface `**tap-r**` is assigned to the `**DHCP-R**` namespace which is connected to the OpenvSwitch `**OVS1**` with VLAN ID 100.
-   TAP interface `**tap-g**` is assigned to the `**DHCP-G**` namespace which is connected to the OpenvSwitch `**OVS1**` with VLAN ID 200.
-   veth interface `**veth-r**`and `**veth-g**`are connected to the `**OVS1**` **OpenVswitch** with VLAN TAG 100 and 200.
-   Interface `**eth0-r**` in the `**red**` namespace receives IP from the `**DNSMASQ DHCP server**` running in the `**DHCP-R**` namespace through the `**tap-r**` network interface.
-   Interface `**eth0-g**` in the `**green**` namespace receives IP from the `**DNSMASQ DHCP server**` running in the `**DHCP-G**` namespace through the `**tap-g**` interface.

**Prerequisites for this this LAB**  
— **Linux system** → For this lab, I used a **Ubuntu** system but any Linux distribution can be used to complete this exercise.  
\- Installation of **OpenVswitch**

1.Create the network namespaces **red**, **green**, **dhcp**\-**r** and **dhcp**\-**g** and the veth pairs `**eth0-r → veth-r**` and `**eth0-g → veth-g**`.

\# ip netns add red  
\# ip netns add green  
\# ip netns add dhcp-r  
\# ip netns add dhcp-g  
\# ip link add eth0-r type veth peer name veth-r  
\# ip link add eth0-g type veth peer name veth-g

2\. Move the `**eth0-r**` veth interface to the `**red**` namespace and then move `**eth0-g**` veth interface to the `**green**` namespace. Then bring up these devices.

\# ip link set eth0-r netns red  
\# ip link set eth0-g netns green  
\# ip netns exec red ip link set dev lo up  
\# ip netns exec red ip link set dev eth0-r up  
\# ip netns exec green ip link set dev lo up  
\# ip netns exec green ip link set dev eth0-g up

3. Connect `**veth-r**` and `**veth-g**` veth interfaces to the `**OVS1**` switch and then assign **vlan** 100 to the `**veth-r**` and the vlan tag 200 to the `**veth-g**` network interface. Then bring up these devices.

\# ovs-vsctl add-port OVS1 veth-r  
\# ovs-vsctl add-port OVS1 veth-g  
\# ovs-vsctl set port veth-r tag=100  
\# ovs-vsctl set port veth-r tag=200  
\# ip link set dev veth-r up  
\# ip link set dev veth-g up

4. Now create the tap interfaces for the `**DHCP-R**` and `**DHCP-G**` namespace.

\# Configuration for the dhcp-r namespace   
\# ovs-vsctl add-port OVS1 tap-r   
\# ovs-vsctl set interface tap-r type=internal  
\# ip link set tap-r netns dhcp-r  
\# ovs-vsctl set port tap-r tag=100  
\# ip netns exec dhcp-r ip link set dev lo up  
\# ip netns exec dhcp-r ip link set dev tap-r up  
\# Configuration for the dhcp-g namespace  
\# ovs-vsctl add-port OVS1 tap-g   
\# ovs-vsctl set interface tap-g type=internal  
\# ip link set tap-g netns dhcp-g  
\# ovs-vsctl set port tap-g tag=200  
\# ip netns exec dhcp-g ip link set dev lo up  
\# ip netns exec dhcp-g ip link set dev tap-g up

5. Configure the interfaces in the `**dhcp-r**` and `**dhcp-g**` namespaces. Assign the IP `**192.168.50.2/24**` to the `**tap-r**` veth interface inside the `**dhcp-r**` namespace and assign the same IP to the `**tap-g**` interface inside the `**dhcp-g**` namespace.

\# ip netns exec dhcp-r bash  
\# ip address add 10.50.50.2/24 dev tap-r  
\# ip netns exec dhcp-g bash  
\# ip address add 10.50.50.2/24 dev tap-g

6. Start `**DNSMASQ**` DHCP server in the `**dhcp-r**` and `**dhcp-g**` namespaces.

\# ip netns exec dhcp-r dnsmasq --interface=tap-r --dhcp-range=10.50.50.10,10.50.50.100,255.255.255.0   
\# ip netns exec dhcp-g dnsmasq --interface=tap-g --dhcp-range=10.50.50.10,10.50.50.100,255.255.255.0   
\# ps -ef | grep dnsmasq  
nobody      2168       1  0 09:18 ?        00:00:00 dnsmasq --interface=tap-r --dhcp-range=10.50.50.10,10.50.50.100,255.255.255.0

nobody      2171       1  0 09:18 ?        00:00:00 dnsmasq --interface=tap-g --dhcp-range=10.50.50.10,10.50.50.100,255.255.255.0  
\# ip netns identify 2168   
dhcp-r  
\# ip netns identify 2171  
dhcp-g

7. Start `**dhclient**` dhcp client program in the `**red**` and `**green**` namespaces to get IP from the DHCP server. Here the DHCP server running in the dhcp-r namespace provided the IP address `**10.50.50.71/24**` to the `**eth0-r**` veth interface. Similarly the veth interface `**eth0-g**` received the IP address `**10.50.50.39/24**` from the DHCP server running inside `**dhcp-g**` network namespace.

\# ip netns exec red dhclient eth0-r  
\# ip netns exec red ip a  
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
6: eth0-r@if5: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000  
    link/ether 4a:be:26:7f:31:7c brd ff:ff:ff:ff:ff:ff link-netnsid 0  
    inet 10.50.50.71/24 brd 10.50.50.255 scope global dynamic eth0-r  
       valid\_lft 3590sec preferred\_lft 3590sec  
    inet6 fe80::48be:26ff:fe7f:317c/64 scope link   
       valid\_lft forever preferred\_lft forever  
\# ip netns exec green dhclient eth0-g  
\# ip netns exec green ip a  
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

8: eth0-g@if7: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000  
    link/ether 5e:51:37:dd:f4:da brd ff:ff:ff:ff:ff:ff link-netnsid 0  
    inet 10.50.50.39/24 brd 10.50.50.255 scope global dynamic eth0-g  
       valid\_lft 3585sec preferred\_lft 3585sec  
    inet6 fe80::5c51:37ff:fedd:f4da/64 scope link   
       valid\_lft forever preferred\_lft forever

8. Now test the network connectivity to the DHCP server IP address from the red and green namespaces.

root@linux:~# ip netns exec red bash  
root@linux:~# ping 10.50.50.2  
PING 10.50.50.2 (10.50.50.2) 56(84) bytes of data.  
64 bytes from 10.50.50.2: icmp\_seq=1 ttl=64 time=2.89 ms  
64 bytes from 10.50.50.2: icmp\_seq=2 ttl=64 time=0.312 ms  
64 bytes from 10.50.50.2: icmp\_seq=3 ttl=64 time=0.097 ms  
^C  
\--- 10.50.50.2 ping statistics ---  
3 packets transmitted, 3 received, 0% packet loss, time 2022ms  
rtt min/avg/max/mdev = 0.097/1.099/2.889/1.268 ms  
\# ip netns exec green bash  
\# ping 10.50.50.2  
PING 10.50.50.2 (10.50.50.2) 56(84) bytes of data.  
64 bytes from 10.50.50.2: icmp\_seq=1 ttl=64 time=2.97 ms  
64 bytes from 10.50.50.2: icmp\_seq=2 ttl=64 time=0.210 ms  
^C  
\--- 10.50.50.2 ping statistics ---  
2 packets transmitted, 2 received, 0% packet loss, time 1005ms  
rtt min/avg/max/mdev = 0.210/1.591/2.973/1.381 ms

## Network namespace use case — 5

![[Raw/Notes/Raw/Media/Resources/f29f89c7a22cbf73ea97322e3ebe90bc_MD5.png]]

> Create a bridge network(`**vnet-br0**`) using Linux internal bridge utility, create two network namespaces **red**, and **green**. Create two VETH pairs for the red and green namespace, connect one end of the veth pair to the specific namespace and another end to the internal bridge(`**vnet-br0**`). Make sure that the interface in the `**red**` and `**green**` namespace can communicate with the internal and external networks.

**Prerequisites for this this LAB  
**— **Linux system** → For this lab, I used a **Ubuntu** system but any Linux distribution can be used to complete this exercise.

1.Create the namespace red and green then create the network bridge `**vnet-br0**`.

\# ip netns add red   
\# ip netns add green  
\# ip link add vnet-br0 type bridge  
\# ip link

2. Create VETH pairs `**eth0-r → veth-r**` and `**eth0-g → veth-g**` and then move the veth interface `**eth0-r**` to the `**red**` namespace and `**eth0-g**` to the green namespace and then connect the other end of the veth pairs to the `**vnet-br0**` bridge.

  
  
  
  

  
7: vnet-br0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default qlen 1000  
    link/ether 1e:1f:4f:3b:19:50 brd ff:ff:ff:ff:ff:ff

  
8: veth-r@if9: <BROADCAST,MULTICAST> mtu 1500 qdisc noop master vnet-br0 state DOWN mode DEFAULT group default qlen 1000  
    link/ether ea:1d:fc:9a:1e:ce brd ff:ff:ff:ff:ff:ff link\-netns red  
10: veth-g@if11: <BROADCAST,MULTICAST> mtu 1500 qdisc noop master vnet-br0 state DOWN mode DEFAULT group default qlen 1000  
    link/ether 66:27:a3:bc:58:4e brd ff:ff:ff:ff:ff:ff link\-netns green

3. Now bring up the necessary network interfaces.

\# ip link set vnet-br0 up  
\# ip link set veth-r up  
\# ip link set veth-g up  
\# ip netns exec red ip link set lo up  
\# ip netns exec red ip link set eth0-r up  
\# ip netns exec green ip link set lo up  
\# ip netns exec green ip link set eth0-g up  
\# ip link   
7: vnet-br0: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000  
    link/ether 1e:1f:4f:3b:19:50 brd ff:ff:ff:ff:ff:ff  
8: veth-r@if9: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue master vnet-br0 state UP mode DEFAULT group default qlen 1000  
    link/ether ea:1d:fc:9a:1e:ce brd ff:ff:ff:ff:ff:ff link-netns red  
10: veth-g@if11: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue master vnet-br0 state UP mode DEFAULT group default qlen 1000  
    link/ether 66:27:a3:bc:58:4e brd ff:ff:ff:ff:ff:ff link-netns green

4. Configure IP address `**192.168.20.2/24**` to the `**eth0-r**` interfaces in the `**red**` namespace and configure the IP address `**192.168.20.3/24**` to `**eth0-g**` in the `**green**` namespace.

\# ip netns exec red bash  
\# ip address add 192.168.20.2/24 dev eth0-r  
\# ip r  
192.168.20.0/24 dev eth0-r proto kernel scope link src 192.168.20.2   
\# ip a  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
    inet 127.0.0.1/8 scope host lo  
       valid\_lft forever preferred\_lft forever  
    inet6 ::1/128 scope host   
       valid\_lft forever preferred\_lft forever  
9: eth0-r@if8: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000  
    link/ether 4a:be:26:7f:31:7c brd ff:ff:ff:ff:ff:ff link-netnsid 0  
    inet 192.168.20.2/24 scope global eth0-r  
       valid\_lft forever preferred\_lft forever  
    inet6 fe80::48be:26ff:fe7f:317c/64 scope link   
       valid\_lft forever preferred\_lft forever  
\# ip netns exec green bash  
\# ip address add 192.168.20.3/24 dev eth0-g  
\# ip a  
1: lo: <LOOPBACK,UP,LOWER\_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
    inet 127.0.0.1/8 scope host lo  
       valid\_lft forever preferred\_lft forever  
    inet6 ::1/128 scope host   
       valid\_lft forever preferred\_lft forever  
11: eth0-g@if10: <BROADCAST,MULTICAST,UP,LOWER\_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000  
    link/ether 5e:51:37:dd:f4:da brd ff:ff:ff:ff:ff:ff link-netnsid 0  
    inet 192.168.20.3/24 scope global eth0-g  
       valid\_lft forever preferred\_lft forever  
    inet6 fe80::5c51:37ff:fedd:f4da/64 scope link   
       valid\_lft forever preferred\_lft forever  
\# ip r  
192.168.20.0/24 dev eth0-g proto kernel scope link src 192.168.20.3

5. Now network communication is working within the namespace using the `**ping**` command.

\# ip netns exec green bash  
\# ping -c 2 192.168.20.2  
PING 192.168.20.2 (192.168.20.2) 56(84) bytes of data.  
64 bytes from 192.168.20.2: icmp\_seq=1 ttl=64 time=0.382 ms  
64 bytes from 192.168.20.2: icmp\_seq=2 ttl=64 time=0.240 ms  
\# ip netns exec red bash  
\# ping -c 2 192.168.20.3  
PING 192.168.20.3 (192.168.20.3) 56(84) bytes of data.  
64 bytes from 192.168.20.3: icmp\_seq=1 ttl=64 time=0.382 ms  
64 bytes from 192.168.20.3: icmp\_seq=2 ttl=64 time=0.240 ms

6. Assign the IP `**192.168.20.1/24**` to the `**vnet-br0**` bridge interface in the `**root**` network namespace to allow external communication from the `**red**` and `**green**` namespaces.

\# ip address add 192.168.20.1/24 dev vnet-br0

7. Configure `**192.168.20.1**` as default router in the `**green**` and `**red**` namespace.

\# ip netns exec red bash  
\# route add default gw 192.168.20.1  
\# ip netns exec green bash  
\# route add default gw 192.168.20.1

8. Again for external communication we have to add a NAT **MASQUERADE** IPTABLE rule on the `**root**` namespace.

\# iptables -s 192.168.20.0/24 -t nat -A POSTROUTING -j MASQUERADE

9. Now ping to other IP addresses in the **root** namespace is working.

\# ping 192.168.205.8  
PING 192.168.205.8 (192.168.205.8) 56(84) bytes of data.  
64 bytes from 192.168.205.8: icmp\_seq=1 ttl=64 time=0.222 ms  
64 bytes from 192.168.205.8: icmp\_seq=2 ttl=64 time=0.182 ms

However ping to **external** network IP address `**8.8.8.8**` is not working because ipv4 **forwarding** is not enabled on the host system

\# ping 8.8.8.8  
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.  
^C  
\--- 8.8.8.8 ping statistics ---  
3 packets transmitted, 0 received, 100% packet loss, time 2054ms

10. Enable IPV4 forwarding on the host system to allow external communication.

\# sysctl -w net.ipv4.ip\_forward=1  
net.ipv4.ip\_forward = 1

Now the private namespaces `**red**` and `**green**` are able to ping to the external IP address `**8.8.8.8**`.

\# ping 8.8.8.8  
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.  
64 bytes from 8.8.8.8: icmp\_seq=1 ttl=116 time=17.6 ms