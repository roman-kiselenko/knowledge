---
title: IP and pod allocations in EKS
source: https://itnext.io/ip-and-pod-allocations-in-eks-5be6612b8325
clipped: 2023-09-04
published: 
category: aws
tags:
  - k8s
  - aws
  - network
read: false
---

![[Raw/Media/Resources/8fa369ac8a30eac5f5bf40b6e2d558d7_MD5.png]]

When running an EKS cluster, you might face two issues:

-   Running out of IP addresses assigned to pods.
-   Low pod count per node (due to ENI limits).

In this article, you will learn how to overcome those.

Before we start, here is some background on how intra-node networking works in Kubernetes.

When a node is created, the kubelet delegates:

1.  Creating the container to the Container Runtime.
2.  Attaching the container to the network to the CNI.
3.  Mounting volumes to the CSI.

![[Raw/Media/Resources/9831fe1ba6cf259efdcc2aa23814fc19_MD5.png]]

*Let’s focus on the CNI part.*

**Each pod has its own isolated Linux network namespace and is attached to a bridge.**

The CNI is responsible for creating the bridge, assigning the IP and connecting veth0 to the cni0.

![[Raw/Media/Resources/84b35e8428ab43cc8dd1b4bf3895b27e_MD5.png]]

This usually happens, but different CNIs might use other means to connect the container to the network.

**As an example, there might not be a cni0 bridge.**

The AWS-CNI is an example of such a CNI.

![[Raw/Media/Resources/f2f52c5d094619e4c86220988a8f1e02_MD5.png]]

In AWS, each EC2 instance can have multiple network interfaces (ENIs).

**You can assign a limited number of IPs to each ENI.**

For example, an `m5.large` can have up to 10 IPs for ENI.

Of those 10 IPs, you have to assign one to the network interface.

The rest you can give away.

![[Raw/Media/Resources/93aaeec463a35c3001451b2869b8a6c1_MD5.png]]

**Previously, you could use the extra IPs and assign them to Pods.**

But there was a big limit: the number of IP addresses.

*Let’s have a look at an example.*

With an `m5.large`, you have up to 3 ENIs with 10 IP private addresses each.

Since one IP is reserved, you’re left with 9 per ENI (or 27 in total).

That means that your `m5.large` could run up to 27 Pods.

*Not a lot.*

![[Raw/Media/Resources/7e9159fcad19ff3738d2405c5ac427a7_MD5.png]]

**But AWS released a change to EC2 that allows “prefixes” to be assigned to network interfaces.**

*Prefixes what?!*

In simple words, ENIs now support a range instead of a single IP address.

**If before you could have 10 private IP addresses, now you can have 10 slots of IP addresses.**

*And how big is the slot?*

By default, 16 IP addresses.

With 10 slots, you could have up to 160 IP addresses.

That’s a rather significant change!

*Let’s have a look at an example.*

![[Raw/Media/Resources/7be310198a5cb5b33af8f9ea0088532b_MD5.png]]

With an `m5.large`, you have 3 ENIs with 10 slots (or IPs) each.

Since one IP is reserved for the ENI, you’re left with 9 slots.

Each slot is 16 IPs, so `9*16=144` IPs.

Since there are 3 ENIs, `144x3=432` IPs.

**You can have up to 432 Pods now (vs 27 before).**

![[Raw/Media/Resources/b74b51c3076bf64c34a658d1d37f770c_MD5.png]]

The AWS-CNI support slots and caps the max number of Pods to 110 or 250, so you won’t be able to run 432 Pods on an `m5.large`.

It’s also worth pointing out that this is not enabled by default — not even in newer clusters.

*Perhaps because only nitro instances support it.*

Assigning slots it’s great until you realize that the CNI gives 16 IP addresses at once instead of only 1, which has the following implications:

-   Quicker IP space exhaustion.
-   Fragmentation.

*Let’s review those.*

![[Raw/Media/Resources/127003958070efb0e649b414c9cb808a_MD5.png]]

A pod is scheduled to a node.

The AWS-CNI allocates 1 slot (16 IPs), and the pod uses one.

Now imagine having 5 nodes and a deployment with 5 replicas.

*What happens?*

![[Raw/Media/Resources/f3669f47019745e235013a580283596a_MD5.png]]

**The Kubernetes scheduler prefers to spread the pods across the cluster.**

Likely, each node receives 1 pod, and the AWS-CNI allocates 1 slot (16 IPs).

You allocated `5*15=75` IPs from your network, but only 5 are used.

![[Raw/Media/Resources/b2646404cdf8dde8b18f3c1e82749b65_MD5.png]]

*But there’s more.*

**Slots allocate a contiguous block of IP addresses.**

If a new IP is assigned (e.g. a node is created), you might have an issue with fragmentation.

![[Raw/Media/Resources/74e843c60867145f9bb3c4be4294c885_MD5.png]]

*How can you solve those?*

-   [You can assign a secondary CIDR to EKS.](https://aws.amazon.com/premiumsupport/knowledge-center/eks-multiple-cidr-ranges/)
-   [You can reserve IP space within a subnet for exclusive use by slots.](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-cidr-reservation.html)

Relevant links:

-   [https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI)
-   [https://aws.amazon.com/blogs/containers/amazon-vpc-cni-increases-pods-per-node-limits/](https://aws.amazon.com/blogs/containers/amazon-vpc-cni-increases-pods-per-node-limits/)
-   [https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-prefix-eni.html#ec2-prefix-basics](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-prefix-eni.html#ec2-prefix-basics)

And finally, if you’ve enjoyed this thread, you might also like:

-   The Kubernetes workshops that we run at Learnk8s [https://learnk8s.io/training](https://learnk8s.io/training)
-   This collection of past threads [https://twitter.com/danielepolencic/status/1298543151901155330](https://twitter.com/danielepolencic/status/1298543151901155330)
-   The Kubernetes newsletter I publish every week [https://learnk8s.io/learn-kubernetes-weekly](https://learnk8s.io/learn-kubernetes-weekly)