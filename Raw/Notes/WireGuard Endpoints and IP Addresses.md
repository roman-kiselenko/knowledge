---
title: WireGuard Endpoints and IP Addresses
source: https://www.procustodibus.com/blog/2021/01/wireguard-endpoints-and-ip-addresses/
clipped: 2023-09-04
published: 2021-01-02
category: development
tags:
  - wireguard
read: true
---

When getting started with WireGuard, it can be hard to understand the interaction between the network layers below WireGuard (the “real” network, often a physical Ethernet or WiFi network) and the WireGuard VPN (Virtual Private Network). This article will cover which configuration settings are used for which (“real” network or virtual network), and will trace how an individual packet flows between the two.

If you’re wondering about some of the terminology used by this article, take a look at at the [WireGuard Terminology](https://www.procustodibus.com/blog/2020/12/wireguard-terminology/) article for clarification.

## IP Address Settings

Let’s first look at a simple WireGuard configuration file. Here you’ll see three different settings with IP addresses — Address, AllowedIPs, and Endpoint:

\# /etc/wireguard/wg0.conf

\[Interface\]
PrivateKey = AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEE=
Address = 10.0.0.1/32
ListenPort = 51821

\[Peer\]
PublicKey = fE/wdxzl0klVp/IR8UcaoGUMjqaWi3jAd7KzHKFS6Ds=
AllowedIPs = 192.168.200.0/24
Endpoint = 203.0.113.2:51822

### Address

The **Address** setting is the virtual address of the local WireGuard peer. It’s the IP address of the virtual network interface that WireGuard sets up for the peer; and as such you can set it to whatever you want (whatever makes sense for the virtual WireGuard network you’re building).

Like with other network interfaces, the IP address for a WireGuard interface is defined with a network prefix, which tells the local host what other IP addresses are available on the same virtual subnet as the interface. In the above example, this prefix is /32 (which generally is a safe default for a WireGuard interface). If we set it to /24, that would indicate to the local host that other addresses in the same /24 block as the address itself (10.0.0.0 to 10.0.0.255) are routable through the interface.

### AllowedIPs

But WireGuard doesn’t use this network prefix to govern what is *actually* routed through the interface — WireGuard instead relies on the **AllowedIPs** setting, configured separately for each peer to which the interface can connect. WireGuard will drop any traffic routed to the interface that has a destination address outside of the AllowedIPs configured for the interface’s peers, and will also drop any traffic coming into the host through the interface that has a source address outside of those same AllowedIPs.

If you use the wg-quick program (or the WireGuard app on platforms like iOS, Android, etc) to set up a WireGuard interface, this program will also configure the host’s routing table with routes that match the AllowedIPs setting, so that traffic which *can* be routed to the WireGuard interface *will* be routed to it.

If you wanted to route all a host’s traffic through a WireGuard interface, you’d configure the interface with one peer, and set that peer’s AllowedIPs to be 0.0.0.0/0, ::/0. In the above example, however, we want to route just a particular subnet to the WireGuard interface — a particular internal site we want to be able to access through a WireGuard tunnel to a peer that’s located in the site — so so we set AllowedIPs for the peer to 192.168.200.0/24 (the block of addresses from 192.168.200.0 to 192.168.200.255).

If we also wanted this interface to be able to route to the 10.0.0.0/24 subnet, regardless of what we set for the interface’s Address setting, we’d have to explicitly add that subnet to the AllowedIPs setting for one of the interface’s peers. (Or more likely, apply different blocks of the subnet to the AllowedIPs setting of different peers, if the interface has multiple peers.)

Note that you can specify multiple IP addresses (or blocks of addresses) either by separating them with commas in an individual AllowedIPs setting, or you can just specify the AllowedIPs setting multiple times for the same peer:

\[Peer\]
PublicKey = fE/wdxzl0klVp/IR8UcaoGUMjqaWi3jAd7KzHKFS6Ds=
AllowedIPs = 192.168.200.0/24, 10.0.0.0/24
# or alternately:
AllowedIPs = 192.168.200.0/24
AllowedIPs = 10.0.0.0/24

As you can see, IP addresses specified by the AllowedIPs setting can be a mix of “real” addresses (like a “real” remote subnet) and virtual addresses (like the other addresses in your WireGuard VPN).

### Endpoint

When traffic is routed to a virtual WireGuard interface, WireGuard needs to know where to send that traffic on a “real” network. The **Endpoint** setting for each peer tells WireGuard the “real” IP address and port to which it should ultimately send traffic.

In the original example above, the peer specified for the interface has an AllowedIPs setting of 192.168.200.0/24, and an Endpoint setting of 203.0.113.2:51822. This means that for any traffic routed to the interface within an IP address in the range of 192.168.200.0 to 192.168.200.255, WireGuard will encrypt and reroute the traffic over a “real” network interface to the “real” remote address of 203.0.113.2 (at UDP port 51822).

Note that you only need to configure a static Endpoint setting on one side of a WireGuard connection. If, on Peer A, we configure an Endpoint setting for Peer B, we can skip configuring an Endpoint setting on Peer B for Peer A — Peer B will wait until Peer A connects to it, and then dynamically update its Endpoint setting to the actual IP address and port from which Peer A connected.

## Packet Flow

Now let’s walk through a concrete example where we trace the flow of packets from one network endpoint to another, and back again. We’ll use a [Point to Site](https://www.procustodibus.com/blog/2020/10/wireguard-topologies/#point-to-site) topology, where we have an “Endpoint A”, an end-user tablet running WireGuard, connected to “Host Beta”, also running WireGuard, and serving as a router for “Site B”. Within Site B, we’ll have “Endpoint B”, an HTTP server to which Endpoint A will try to send some traffic.

The diagram below illustrates this scenario:

Point-to-site scenario (Endpoint A to Site B)

![WireGuard point-to-site scenario](https://www.procustodibus.com/images/blog/wireguard-topologies/point-to-site-complex.svg)

The configuration for this scenario is explained in the [Point to Site Configuration](https://www.procustodibus.com/blog/2020/11/wireguard-point-to-site-config/) article. These are the WireGuard configuration settings we’d use for Endpoint A:

\# local settings for Endpoint A
\[Interface\]
PrivateKey = AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEE=
Address = 10.0.0.1/32
ListenPort = 51821

# remote settings for Host β
\[Peer\]
PublicKey = fE/wdxzl0klVp/IR8UcaoGUMjqaWi3jAd7KzHKFS6Ds=
AllowedIPs = 192.168.200.0/24
Endpoint = 203.0.113.2:51822

And the configuration settings we’d use for Host Beta:

\# local settings for Host β
\[Interface\]
PrivateKey = ABBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBFA=
Address = 10.0.0.2/32
ListenPort = 51822

# IP forwarding
PreUp = sysctl -w net.ipv4.ip\_forward=1
# IP masquerading (source NAT)
PreUp = iptables -t mangle -A PREROUTING -i wg0 -j MARK --set-mark 0x30
PreUp = iptables -t nat -A POSTROUTING ! -o wg0 -m mark --mark 0x30 -j MASQUERADE
PostDown = iptables -t mangle -D PREROUTING -i wg0 -j MARK --set-mark 0x30
PostDown = iptables -t nat -D POSTROUTING ! -o wg0 -m mark --mark 0x30 -j MASQUERADE

# remote settings for Endpoint A
\[Peer\]
PublicKey = /TOE4TKtAqVsePRVR+5AA43HkAK5DSntkOCO7nYq5xU=
AllowedIPs = 10.0.0.1/32

### Endpoint A

Endpoint A has a WiFi network interface named wlan0, with an IP address of 192.168.1.11; and a WireGuard interface named wg0, with an IP address of 10.0.0.1 (the WireGuard interface shown in the above configuration).

To kick things off, on Endpoint A the user navigates to http://192.168.200.22/ in a web browser (192.168.200.22 is the IP address of Endpoint B in Site B.) The web browser will use a high-level API to try to open a TCP socket to port 80 of 192.168.200.22. The network stack of the OS (Operating System) running on Endpoint A will implement this in part by sending off a few TCP packets (the first of which we’ll trace).

Before sending off any packets, however, the OS on Endpoint A will check its routing tables to determine which local network interface to bind the new TCP socket to. Since we configured the AllowedIPs setting on Endpoint A’s wg0 interface to 192.168.200.0/24 (and used wg-quick to manage this interface), WireGuard added an entry to Endpoint A’s routing table that instructs it to use wg0 to connect to the 192.168.200.0/24 block of addresses.

Therefore, since it’s supposed to connect to 192.168.200.22, Endpoint A will select wg0 as the interface for this TCP socket, and use the wg0 interface’s IP address of 10.0.0.1 for the source address of the socket. For the source port, it will select a random, unused port on 10.0.0.1; in our case, let’s say it selects port 50000.

Then to initiate this TCP socket, the OS on Endpoint A will generate a SYN TCP packet, and queue it on its wg0 interface:

         

Step

Host

Interface

Direction

Protocol

Source

Port

Destination

Port

Content

1

Endpoint A

wg0

out

TCP

10.0.0.1

50000

192.168.200.22

80

SYN

When the wg0 interface on Endpoint A pulls this packet from its queue, it will check its table of configured peers to see which peer, if any, has an AllowedIPs setting that includes the packet’s destination address of 192.168.200.22. In our case, we have only one peer configured for the interface, and its AllowedIPs setting matches. Therefore, WireGuard will encrypt the original TCP packet using the public key for the peer, and wrap it in a new UDP packet that uses the peer’s Endpoint setting as the new packet’s destination address and port (203.0.113.2:51822).

WireGuard will also use the host’s routing tables to determine what network interface and IP address to use to send out this new UDP packet. In our case, Endpoint A’s only “real” network interface is wlan0, so it will use that interface to send out the packet; and it will use the wlan0 interface’s only IP address, 192.168.1.11 as the packet’s source address. For source port, it will use the wg0 interface’s ListenPort configuration setting (51821, as shown in the above configuration):

         

Step

Host

Interface

Direction

Protocol

Source

Port

Destination

Port

Content

2

Endpoint A

wlan0

out

UDP

192.168.1.11

51821

203.0.113.2

51822

encrypted data

### NAT Router

The UDP packet will make it’s way to the NAT (Network Address Translation) router in front of Endpoint A, where the router will translate the source address of the packet to the router’s own public-facing IP address, as well as change the source port to something the router will remember is associated with Endpoint A. In our example, this IP address is 198.51.100.1, and we’ll say the port will be 51111:

         

Step

Host

Interface

Direction

Protocol

Source

Port

Destination

Port

Content

3

Site A NAT Router

wlan0

in

UDP

192.168.1.11

51821

203.0.113.2

51822

encrypted data

4

Site A NAT Router

eth0

out

UDP

198.51.100.1

51111

203.0.113.2

51822

encrypted data

The NAT router will then send the packet out to the Internet, where eventually it will make its way to Host Beta.

### Host B

Host Beta has two Ethernet network interfaces, eth0, with an IP address of 203.0.113.2, connected to the Internet; and eth1, with an IP address of 192.168.200.2, connected to Site B’s LAN (Local Area Network). It also has a WireGuard interface named wg0, with an IP address of 10.0.0.2 (as shown in the configuration at the top of this packet-flow section). It’s also configured to NAT any traffic it receives inbound to its wg0 interface.

Note the NAT for this wg0 interface is opposite of the way Site B’s traffic normally would be NATed to the Internet. Normally outbound traffic from Site B to the Internet would be NATed to rewrite the source address of all packets originating in Site B to use its Internet gateway’s external address (and rewrite the destination address of packets coming back into Site B in response). But for wg0, we’re treating the WireGuard VPN as the “local site”, and instead NATing traffic outbound from it to Site B. This is needed if the hosts in Site B wouldn’t otherwise know to route traffic for the WireGuard VPN through Host Beta — but unnecessary if they do.

So our UDP packet, originally from Endpoint A (by way of the NAT router in Site A and the Internet), will come into Host Beta on eth0:

         

Step

Host

Interface

Direction

Protocol

Source

Port

Destination

Port

Content

5

Host Beta

eth0

in

UDP

198.51.100.1

51111

203.0.113.2

51822

encrypted data

As configured by the wg0 interface’s ListenPort setting, WireGuard will be listening on UDP port 51822 of all Host Beta’s network interfaces for such packets. WireGuard will decrypt the data from this packet, and whatever packets it finds in the decrypted data, it will requeue on the host’s network stack as if they had come in directly from the wg0 interface on Host Beta:

         

Step

Host

Interface

Direction

Protocol

Source

Port

Destination

Port

Content

6

Host Beta

wg0

in

TCP

10.0.0.1

50000

192.168.200.22

80

SYN

The OS on Host Beta will check its routing tables, and see that a packet destined for 192.168.200.22 should be forwarded out its eth1 network interface. And since Host Beta is also configured to NAT traffic that originated from wg0, it will translate the packet’s source address to the IP address it uses for eth1 (192.168.200.2), and choose a random port to associate with the original source address (say 52222):

         

Step

Host

Interface

Direction

Protocol

Source

Port

Destination

Port

Content

7

Host Beta

eth1

out

TCP

192.168.200.2

52222

192.168.200.22

80

SYN

### Endpoint B

Endpoint B has an Ethernet network interface named eth0, with an IP address of 192.168.200.22. It has an HTTP server listening on TCP port 80 of this interface, so it accepts the TCP SYN packet when it receives it from Host Beta:

         

Step

Host

Interface

Direction

Protocol

Source

Port

Destination

Port

Content

8

Endpoint B

eth0

in

TCP

192.168.200.2

52222

192.168.200.22

80

SYN

And in response, it generates a SYN-ACK packet that reverses the source and destination of the original SYN packet, and sends that packet to Host Beta:

         

Step

Host

Interface

Direction

Protocol

Source

Port

Destination

Port

Content

9

Endpoint B

eth0

out

TCP

192.168.200.22

80

192.168.200.2

52222

SYN-ACK

### Host B Return

Host Beta receives this SYN-ACK packet, and sees that it’s sent to a port (52222) that it had recently used for NAT:

         

Step

Host

Interface

Direction

Protocol

Source

Port

Destination

Port

Content

10

Host Beta

eth1

in

TCP

192.168.200.22

80

192.168.200.2

52222

SYN-ACK

So it looks up that port in its NAT tables, and translates the destination address back to 10.0.0.1, and the destination port back to 50000. It checks its routing tables for 10.0.0.1, and sees that it’s to be routed out its wg0 interface, so it queues the packet to go out that interface:

         

Step

Host

Interface

Direction

Protocol

Source

Port

Destination

Port

Content

11

Host Beta

wg0

out

TCP

192.168.200.22

80

10.0.0.1

50000

SYN-ACK

When the wg0 interface pulls the packet out of its queue, WireGuard will check its table of configured peers to see which peer has an AllowedIPs setting that includes the packet’s destination address 10.0.0.1. The peer configured for Endpoint A, with its AllowedIPs setting of 10.0.0.1/32, matches; so WireGuard will encrypt the packet using the public key for Endpoint A.

Since no Endpoint is configured on Host Beta for the Endpoint A peer, WireGuard will use the IP address and port from the last packet it received from the Endpoint A peer, which was 198.51.100.1 port 51111. The routing tables on Host Beta tell it to use eth0 to send traffic to 198.51.100.1, and to use a source IP address of 203.0.113.2 with it. And the ListenPort configuration setting for wg0 tells it to use port 51822 as the source of its traffic.

So WireGuard will queue a new UDP packet to be sent by Host Beta out its eth0 interface to 198.51.100.1, containing the encrypted SYN-ACK packet:

         

Step

Host

Interface

Direction

Protocol

Source

Port

Destination

Port

Content

12

Host Beta

eth0

out

UDP

203.0.113.2

51822

198.51.100.1

51111

encrypted data

### NAT Router Return

The NAT router in front of Endpoint A will receive this UDP packet from Host Beta on its public network interface, and will recognize the destination port as a port it’s recently used for NAT. It will translate the destination address and port back to the original address and port used by Endpoint A, and forward the packet on to Endpoint A out the router’s LAN interface:

         

Step

Host

Interface

Direction

Protocol

Source

Port

Destination

Port

Content

13

Site A NAT Router

eth0

in

UDP

203.0.113.2

51822

198.51.100.1

51111

encrypted data

14

Site A NAT Router

wlan0

out

UDP

203.0.113.2

51822

192.168.1.11

51821

encrypted data

### Endpoint A Return

Endpoint A receives the UDP packet from the NAT router on its wlan0 interface:

         

Step

Host

Interface

Direction

Protocol

Source

Port

Destination

Port

Content

15

Endpoint A

wlan0

in

UDP

203.0.113.2

51822

192.168.1.11

51821

encrypted data

And since the ListenPort setting of Endpoint A’s wg0 interface is 51821, WireGuard will be listening for this packet, and will decrypt it. As it contains a TCP packet, and this TCP packet’s source address matches the AllowedIPs setting for the remote peer from which it was received, WireGuard will requeue this decrypted TCP packet on Endpoint A’s wg0 interface:

         

Step

Host

Interface

Direction

Protocol

Source

Port

Destination

Port

Content

16

Endpoint A

wg0

in

TCP

192.168.200.22

80

10.0.0.1

50000

SYN-ACK

Since Endpoint A’s OS was in the process of opening a TCP socket on local port 50000 of the wg0 interface, it handles the packet, sees that it contains the expected SYN-ACK, and completes the TCP handshake by sending a new TCP ACK packet back to Endpoint B.

This ACK packet follows the same exact flow as the original SYN packet. After sending it off, the TCP socket has been established, and the OS on Endpoint A returns the newly-established socket to the web browser from the original API call the browser made. The browser can now send a stream of data to this socket, which the OS will package into TCP packets, and send on to Endpoint B. These packets, too, will follow the same exact flow as the original SYN packet.

### Round Trip

To wrap up, here’s a diagram of the complete round trip of the SYN packet and SYN-ACK response described above:

WireGuard packet round trip

![WireGuard packet flow](https://www.procustodibus.com/images/blog/endpoints/wireguard-packet-flow.svg)

And the combined steps in a single table:

         

Step

Host

Interface

Direction

Protocol

Source

Port

Destination

Port

Content

1

Endpoint A

wg0

out

TCP

10.0.0.1

50000

192.168.200.22

80

SYN

2

Endpoint A

wlan0

out

UDP

192.168.1.11

51821

203.0.113.2

51822

encrypted data

3

Site A NAT Router

wlan0

in

UDP

192.168.1.11

51821

203.0.113.2

51822

encrypted data

4

Site A NAT Router

eth0

out

UDP

198.51.100.1

51111

203.0.113.2

51822

encrypted data

5

Host Beta

eth0

in

UDP

198.51.100.1

51111

203.0.113.2

51822

encrypted data

6

Host Beta

wg0

in

TCP

10.0.0.1

50000

192.168.200.22

80

SYN

7

Host Beta

eth1

out

TCP

192.168.200.2

52222

192.168.200.22

80

SYN

8

Endpoint B

eth0

in

TCP

192.168.200.2

52222

192.168.200.22

80

SYN

9

Endpoint B

eth0

out

TCP

192.168.200.22

80

192.168.200.2

52222

SYN-ACK

10

Host Beta

eth1

in

TCP

192.168.200.22

80

192.168.200.2

52222

SYN-ACK

11

Host Beta

wg0

out

TCP

192.168.200.22

80

10.0.0.1

50000

SYN-ACK

12

Host Beta

eth0

out

UDP

203.0.113.2

51822

198.51.100.1

51111

encrypted data

13

Site A NAT Router

eth0

in

UDP

203.0.113.2

51822

198.51.100.1

51111

encrypted data

14

Site A NAT Router

wlan0

out

UDP

203.0.113.2

51822

192.168.1.11

51821

encrypted data

15

Endpoint A

wlan0

in

UDP

203.0.113.2

51822

192.168.1.11

51821

encrypted data

16

Endpoint A

wg0

in

TCP

192.168.200.22

80

10.0.0.1

50000

SYN-ACK