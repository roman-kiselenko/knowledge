---
title: Kubernetes scheduler deep dive
source: https://itnext.io/kubernetes-scheduler-deep-dive-fdfcb516be30
clipped: 2023-09-04
published: 
category: k8s
tags:
  - k8s
  - scheduler
read: false
---

![[Raw/Media/Resources/1afe1ec506b911a19ad7ea4b57c5990f_MD5.png]]

The scheduler is in charge of deciding where your pods are deployed in the cluster.

It might sound like an easy job, but it’s rather complicated!

Let’s start with the basic.

When you submit a deployment with kubectl, the API server receives the request, and the resource is stored in etcd.

*Who creates the pods?*

![[Raw/Media/Resources/7c54e0d267cb5962f098985c9b21fe67_MD5.png]]

It’s a common misconception that it’s the scheduler’s job to create the pods.

**Instead, the controller manager creates them (and the associated ReplicaSet).**

![[Raw/Media/Resources/909bfc5d531991fc7f39118fb95b82f3_MD5.png]]

At this point, the pods are stored as “Pending” in the etcd and are not assigned to any node.

They are also added to the scheduler’s queue, ready to be assigned.

![[Raw/Media/Resources/8fca4a3a7029bc581fd3897c98e05ead_MD5.png]]

The scheduler process Pods 1 by 1 through two phases:

1.  Scheduling phase (what node should I choose?).
2.  Binding phase (let’s write to the database that this pod belongs to that node).

![[Raw/Media/Resources/61616bbb6f4fb055001aaff4093b78f2_MD5.png]]

The Scheduler phase is divided into two parts. The Scheduler:

1.  Filters relevant nodes (using a list of functions called predicates)
2.  Ranks the remaining nodes (using a list of functions called priorities)

Let’s have a look at an example.

![[Raw/Media/Resources/8f997add5d3316b03ea647414647a70b_MD5.png]]

Consider the following cluster with nodes with and without GPU.

Also, a few nodes are already running at total capacity.

![[Raw/Media/Resources/3ac9c2d3be1bedbf4f663851852dea35_MD5.png]]

You want to deploy a Pod that requires some GPU.

You submit the pod to the cluster, and it’s added to the scheduler queue.

The scheduler discards all nodes that don’t have GPU (filter phase).

![[Raw/Media/Resources/b057805f2fa0f5394038045b24a80c25_MD5.png]]

Next, the scheduler scores the remaining nodes.

In this example, the fully utilized nodes are scored lower.

In the end, the empty node is selected.

![[Raw/Media/Resources/63c9369b72b28b841e7e5b1b461843c4_MD5.png]]

What are some examples of filters?

-   `NodeUnschedulable` prevents pods from landing on nodes marked as unschedulable.
-   `VolumeBinding` checks if the node can bind the requested volume.

The default filtering phase has 13 predicates.

![[Raw/Media/Resources/7c25ee5b330542cf0329207b849f3b5f_MD5.png]]

Here are some examples of scoring:

-   `ImageLocality` prefers nodes that already have the container image downloaded locally.
-   `NodeResourcesBalancedAllocation` prefers underutilized nodes.

There are 13 functions to decide how to score and rank nodes.

![[Raw/Media/Resources/65f3b21533d00e20f4803979506316d6_MD5.png]]

How can you influence the scheduler’s decisions?

-   `nodeSelector`
-   Node affinity
-   Pod affinity/anti-affinity
-   Taints and tolerations
-   Topology constraints
-   Scheduler profiles

`nodeSelector` is the most straightforward mechanism.

You assign a label to a node and add that label to the pod.

The pod can only be deployed on nodes with that label.

![[Raw/Media/Resources/4c168cf486bea2de1b06aebd117f20e1_MD5.png]]

Node affinity extends nodeSelector with a more flexible interface.

You can still tell the scheduler where the Pod should be deployed, but you can also have soft and hard constraints.

![[Raw/Media/Resources/17ef4967b466066bc3c346b4748a61a0_MD5.png]]

With Pod affinity/anti-affinity, you can ask the scheduler to place a pod next to a specific pod.

Or not.

For example, you could have a deployment with anti-affinity on itself to force spreading pods.

![[Raw/Media/Resources/27e3dc24e875e3eb3f16ad34cdd51ca2_MD5.png]]

With taints and tolerations, pods are tainted, and nodes repel (or tolerate) pods.

This is similar to node affinity, but there’s a notable difference: with Node affinity, Pods are attracted to nodes.

Taints are the opposite — they allow a node to repel pods.

![[Raw/Media/Resources/85e95d4f3b69ced805f3516846f09132_MD5.png]]

Moreover, tolerations can repel pods with three effects: evict, “don’t schedule”, and “prefer don’t schedule”.

Personal note: this is one of the most difficult APIs I worked with.

I always (and consistently) get it wrong as it’s hard (for me) to reason in double negatives.

You can use topology spread constraints to control how Pods are spread across your cluster.

This is convenient when you want to ensure that all pods aren’t landing on the same node.

![[Raw/Media/Resources/a05b0bc27518fc7eecd7bf33bf34aab8_MD5.png]]

And finally, you can use Scheduler policies to customize how the scheduler uses filters and predicates to assign nodes to pods.

This relatively new feature (>1.25) allows you to turn off or add new logic to the scheduler.

![[Raw/Media/Resources/4219ec50558c2e61dda430bca3ccd058_MD5.png]]

You can learn more about the scheduler here:

-   Kubernetes scheduler [https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
-   Scheduling framework [https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
-   Scheduler policies [https://kubernetes.io/docs/reference/scheduling/config/](https://kubernetes.io/docs/reference/scheduling/config/)

And finally, if you’ve enjoyed this thread, you might also like:

-   The Kubernetes workshops that we run at Learnk8s [https://learnk8s.io/training](https://learnk8s.io/training)
-   This collection of past threads [https://twitter.com/danielepolencic/status/1298543151901155330](https://twitter.com/danielepolencic/status/1298543151901155330)
-   The Kubernetes newsletter I publish every week “Learn Kubernetes weekly” [https://learnk8s.io/learn-kubernetes-weekly](https://learnk8s.io/learn-kubernetes-weekly)