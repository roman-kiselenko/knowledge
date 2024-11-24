---
title: Linux Networking DNS
source: https://yuminlee2.medium.com/linux-networking-dns-7ff534113f7d
clipped: 2023-09-04
published: 
category: development
tags:
  - network
  - dns
  - low-level
read: false
---

The `/etc/hosts` file on Linux and Unix-based systems **maps hostnames to IP addresses**. It is used by the system’s resolver library to resolve hostnames without consulting a Domain Name System (DNS) server. Each line contains an **IP address followed by one or more hostnames** separated by whitespace. The IP address and hostname mapping is **entered manually by a system administrator**.

**$ cat /etc/hosts**  
\# This is a sample hosts file used by Linux operating systems

\# IP address    hostname  
127.0.0.1       localhost  
127.0.1.1       myhostname.localdomain  myhostname  
192.168.1.100   myserver.example.com  myserver

While editing the `/etc/hosts` file is a simple way to map hostnames to IP addresses on a local system, it becomes **impractical in larger networks** where multiple systems need to access different hosts. Maintaining a large number of entries in the `/etc/hosts` file for each system in the network can be a tedious and error-prone task.

![[Raw/Media/Resources/da977340521e3bdb0be951981fe88e75_MD5.png]]

/etc/hosts file

In these cases, a **centralized DNS server** is often used to provide a **more scalable and flexible solution for hostname resolution**.

Domain Name System(DNS) enables the **translation of human-readable domain names into IP addresses** that machines can use to communicate with each other.

![[Raw/Media/Resources/df5bb7ee67b4b334bae699c1b2f3ca44_MD5.png]]

Domain Name System (DNS)

When a Linux system needs to resolve a domain name, it sends a query to a DNS server. The DNS server then responds with the IP address associated with the domain name. If the DNS server does not have a record for the domain name, it can forward the query to other DNS servers until it finds the correct IP address.

## Configure DNS servers

![[Raw/Media/Resources/a5e0f95389355e636ad98ab0be67dace_MD5.png]]

Configure DNS servers

To configure the DNS server in Linux, we can modify the `**/etc/resolv.conf**` file. This is a **plain text file** that **contains the IP address of the DNS server** that the system should use for hostname resolution. To add a DNS server, we simply need to insert a “**nameserver**” line followed by the IP address of the DNS server.

For instance, to use the Google Public DNS server with IP address 8.8.8.8, we can add the following line to `/etc/resolv.conf`:

nameserver 8.8.8.8

If we want to use multiple DNS servers, we can add additional “nameserver” lines, one for each DNS server, like this:

nameserver 8.8.8.8  
nameserver 8.8.4.4

Once we save the changes to the `/etc/resolv.conf` file, the system will use the specified DNS server(s) for hostname resolution.

## Hostname-to-IP Resolution Order

![[Raw/Media/Resources/833160e2455a511b928a07a5d16fccd7_MD5.png]]

Hostname-to-IP Resolution Order

The order of hostname-to-IP mapping lookup can be specified in the `**/etc/nsswitch.conf**` file. This file **specifies the order of the lookup methods used by the system’s Name Service Switch (NSS) library**.

**By default**, the file includes a line that specifies “**hosts: files dns**”.

\# /etc/nsswitch.conf  
**hosts**:         ** files dns**

> The system first looks in the `/etc/hosts` file for hostname-to-IP mappings, and if the hostname is not found in the file, it queries a DNS server to resolve the hostname.

However, the order can be changed by modifying the “hosts” line in `/etc/nsswitch.conf` to “**hosts: dns files**”.

\# /etc/nsswitch.conf  
**hosts**:          **dns files**

> The system first queries the DNS server for hostname-to-IP mappings, and if the hostname is not found in the DNS server, it looks in the `/etc/hosts` file.

A domain name is **a hierarchical label** used to identify a network domain on the Internet. It consists of multiple levels, separated by dots, with the **rightmost label** representing the **top-level domain (TLD)**, and the other labels representing **subdomains**.

For example, `mail.yahoo.com` is a domain name where `com` is the top-level domain, `yahoo` is the second-level domain and also the subdomain of `com`, and `mail` the third-level domain and a subdomain of `yahoo`.

![[Raw/Media/Resources/e8b3201b62163b2207db012e61fa0499_MD5.png]]

Domain name

> **Top-level domains (TLDs)**: These are the highest level domains in the hierarchical Domain Name System.
> 
> **Second-level domains (SLDs**): These are the domains directly below the TLDs.
> 
> **Subdomains**: These are domains that are **part of a larger domain** and can be created by the owner of the domain.

A search domain is a **domain name** that is **automatically appended to a hostname** when a user tries to access it **without specifying a domain name**. This is particularly useful in situations where there are many hosts on a local network and a user needs to access them frequently. By setting up a search domain, users can simply type the hostname without the domain name, and the system will automatically append the search domain to the hostname and **attempt to resolve the full domain name**.

For example, if the search domain is set to “example.com” and a user wants to connect to a host named “server1”, the system will automatically try to resolve “server1.example.com”.

![[Raw/Media/Resources/5b18e3c38393169b410a01f3eb622fd9_MD5.png]]

Search domain

## Configure search domain

![[Raw/Media/Resources/b1bf46d37b14b49b9d9390006e094fa4_MD5.png]]

Configure search domain

To configure a search domain, we can add a “**search**” line followed by the domain name(s) to the `**/etc/resolv.conf**` file.

For example, to set the search domain to “example.com”, we would add the following line to the file:

\# /etc/resolv.conf  
**search example.com**

We can also specify multiple search domains by separating them with whitespace.

\# /etc/resolv.conf  
**search example.com example.org**

In the Domain Name System (DNS), various types of records are used to associate information with a domain name. The most common types of DNS records are:

![[Raw/Media/Resources/09b3bb94146031975bab79f8d4943810_MD5.png]]

record types

1.  **A Record (Address Record)**: This type of record **maps a domain name to an IPv4 address**.  
    For example, the A record for the domain name “example.com” might point to the IP address “192.0.2.1”. This means that when someone types “example.com” into their web browser, their computer will connect to the server located at the IP address “192.0.2.1”.
2.  **AAAA Record (IPv6 Address Record)**: This type of record **maps a domain name to an IPv6 address**.  
    For example, the AAAA record for the domain name “example.com” might point to the IPv6 address “2001:0db8:85a3:0000:0000:8a2e:0370:7334”. This means that when someone types “example.com” into their web browser, their computer will connect to the server located at the IPv6 address “2001:0db8:85a3:0000:0000:8a2e:0370:7334”.
3.  **CNAME Record (Canonical Name Record)**: This type of record **maps one domain name to another**. It is used to create an **alias** for a domain name.  
    For example, the CNAME record for the domain name “[www.example.com](http://www.example.com/)" might point to the domain name “example.com”. This means that when someone types “[www.example.com](http://www.example.com/)" into their web browser, their computer will connect to the server located at the same IP address as “example.com”.