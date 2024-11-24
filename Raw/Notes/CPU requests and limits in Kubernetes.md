---
title: CPU requests and limits in Kubernetes
source: https://community.ops.io/danielepolencic/cpu-requests-and-limits-in-kubernetes-ock
clipped: 2023-09-04
published: 2023-03-06
category: k8s
tags:
  - k8s
  - pod
  - resources
read: false
---

*In Kubernetes, what should I use as CPU requests and limits?*

Popular answers include:

-   Always use limits!
-   NEVER use limits, only requests!
-   I don't use either; is it OK?

Let's dive into it.

In Kubernetes, you have two ways to specify how much CPU a pod can use:

1.  **Requests** are usually used to determine the average consumption.
2.  **Limits** set the max number of resources allowed.

The Kubernetes scheduler uses requests to determine where the pod should be allocated in the cluster.

Since the scheduler doesn't know the consumption (the pod hasn't started yet), it needs a hint.

But it doesn't end there.

[![[Raw/Media/Resources/95bf7eafec13d3b31f295d84885abbe7_MD5.png]]](https://community.ops.io/images/xGOYmKJryVy5uqknSoZi8XB4yEDMR2Axkl6rzT6KZok/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vbGVhcm5rOHMv/aW1hZ2UvdXBsb2Fk/L3YxNjc4MDc4ODQ1/L3RocmVhZHMvY3B1/LXJlcXVlc3QtbGlt/aXQtM19sd2ZjNzYu/cG5n)

CPU requests are also used to repart the CPU to your containers.

Let's have a look at an example:

-   A node has a single CPU.
-   Container A has requests equal to 0.1 vCPU.
-   Container B has requests equal to 0.2 vCPU.

*What happens when both containers try to use 100% of the available CPU?*

[![[Raw/Media/Resources/354091c53e84edd05ab83367629a18b8_MD5.png]]](https://community.ops.io/images/4CoV3yeqXWfdhObrGMw8mjNeYN9A8jfwA8V2JhJw878/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vbGVhcm5rOHMv/aW1hZ2UvdXBsb2Fk/L3YxNjc4MDc4ODQ1/L3RocmVhZHMvY3B1/LXJlcXVlc3QtbGlt/aXQtNF9qamFmcmIu/cG5n)

Since the CPU request doesn't limit consumption, both containers will use all available CPUs.

However, since container B's request is doubled compared to the other, the final CPU distribution is: **Container 1 uses 0.3vCPU and the other 0.6vCPU (double the amount).**

[![[Raw/Media/Resources/18fafae98db83eb5f526386e27a3e588_MD5.png]]](https://community.ops.io/images/5Y8BkQhq186jZr1AOYYBDceD_bvf0yI-thsmUYsWsqU/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vbGVhcm5rOHMv/aW1hZ2UvdXBsb2Fk/L3YxNjc4MDc4ODQ1/L3RocmVhZHMvY3B1/LXJlcXVlc3QtbGlt/aXQtNV9kcTFqaWMu/cG5n)

Requests are suitable for:

-   Setting a baseline (give me at least X amount of CPU).
-   Setting relationships between pods (this pod A uses twice as much CPU as the other).

But do not help set hard limits.

For that, you need CPU limits.

**When you set a CPU limit, you define a period and quota.**

Example:

-   period: 100000 microseconds (0.1s).
-   quota: 10000 microseconds (0.01s).

I can only use the CPU for 0.01 seconds every 0.1 seconds.

That's also abbreviated as "100m".

[![[Raw/Media/Resources/ef01fc5c751a9f1449815133b5f58481_MD5.png]]](https://community.ops.io/images/C4meBFLCeqdVQ7z4j-aZOX9UQAa6K29pohVloGM4fY0/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vbGVhcm5rOHMv/aW1hZ2UvdXBsb2Fk/L3YxNjc4MDc4ODQ1/L3RocmVhZHMvY3B1/LXJlcXVlc3QtbGlt/aXQtNl92ZDQwZGUu/cG5n)

**If your container has a hard limit and wants more CPU, it has to wait for the next period.**

Your process is throttled.

[![[Raw/Media/Resources/a56a9538059447bc02eb76883b95e5ca_MD5.png]]](https://community.ops.io/images/AjA-kEnSnarZ10GQ2CK10NZiCtIjKho4-y0XQ-LMqAE/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vbGVhcm5rOHMv/aW1hZ2UvdXBsb2Fk/L3YxNjc4MDc4ODQ1/L3RocmVhZHMvY3B1/LXJlcXVlc3QtbGlt/aXQtN19jbnYzamYu/cG5n)

*So what should you use as CPU requests and limits in your Pods?*

A simple (but not accurate) way is to calculate the smallest CPU unit as:  

```
REQUEST = NODE_CORES * 1000 / MAX_NUM_PODS_PER_NODE
```

Enter fullscreen mode Exit fullscreen mode

For a 1 vCPU node and a limit of 10 Pods, that's a `1 * 1000 / 10 = 100Mi` request.

**Assign the smallest unit or a multiplier of it to your containers.**

[![[Raw/Media/Resources/029fe2b24634f13863ff3c40a91ba6bd_MD5.png]]](https://community.ops.io/images/pegycvNGkQIpZW1lr-Upmo9WUTO8o8OpcMyvmfq5ITE/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vbGVhcm5rOHMv/aW1hZ2UvdXBsb2Fk/L3YxNjc4MDc4ODQ1/L3RocmVhZHMvY3B1/LXJlcXVlc3QtbGlt/aXQtOF9jMG5xbTIu/cG5n)

For example, if you don't know how much CPU you need for Pod A, but you identified it is twice as Pod B, you could set:

-   Request A: 1 unit
-   Request B: 2 units

If the containers use 100% CPU, they repart the CPU according to their weights (1:2).

[![[Raw/Media/Resources/c8cd59d5cd740b12869d6cb1c635a1cf_MD5.png]]](https://community.ops.io/images/LUbl8f70lZxYFU9p5Yw_jj7Al4QiZdGi9B9PySDfcNM/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vbGVhcm5rOHMv/aW1hZ2UvdXBsb2Fk/L3YxNjc4MDc4ODQ1/L3RocmVhZHMvY3B1/LXJlcXVlc3QtbGlt/aXQtOV9xenFqYW8u/cG5n)

**A better approach is to monitor the app and derive the average CPU utilization.**

You can do this with your existing monitoring infrastructure or use the [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) to monitor and report the average request value.

[![Vertical Pod Autoscaler with Goldilocks](https://community.ops.io/images/jsOCNaoYRPWkTQY-OVSz88ikTQRJe9_E6kojDICJrpA/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vbGVhcm5rOHMv/aW1hZ2UvdXBsb2Fk/L3YxNjc4MDc4ODQ2/L3RocmVhZHMvZGVt/b19hZ3NpaGsuZ2lm)](https://community.ops.io/images/jsOCNaoYRPWkTQY-OVSz88ikTQRJe9_E6kojDICJrpA/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vbGVhcm5rOHMv/aW1hZ2UvdXBsb2Fk/L3YxNjc4MDc4ODQ2/L3RocmVhZHMvZGVt/b19hZ3NpaGsuZ2lm)

*How should you set the limits?*

1.  Your app might already have "hard" limits. (Node.js is single-threaded and uses up to 1 core even if you assign 2).
2.  You could have: limit = 99th percentile + 30-50%.

You should profile the app (or use the VPA) for a more detailed answer.

[![[Raw/Media/Resources/30846c2820833e1d79bf88c4daedaf70_MD5.png]]](https://community.ops.io/images/4Q0s51_NjvYkYFkazO6igZZAYPS0rMiKfGRiak21Uvw/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vbGVhcm5rOHMv/aW1hZ2UvdXBsb2Fk/L3YxNjc4MDc4ODQ2/L3RocmVhZHMvY3B1/LXJlcXVlc3QtbGlt/aXQtMTBfZWdnNnZi/LnBuZw)

*Should you always set the CPU request?*

**Absolutely, yes.**

This is a standard good practice in Kubernetes and helps the scheduler allocate pods more efficiently.

*Should you always set the CPU limit?*

This is a bit more controversial, but, in general, I think so.

You can find a deeper dive here: [https://dnastacio.medium.com/why-you-should-keep-using-cpu-limits-on-kubernetes-60c4e50dfc61](https://dnastacio.medium.com/why-you-should-keep-using-cpu-limits-on-kubernetes-60c4e50dfc61)

Also, if you want to dig in more a few relevant links:

-   [https://learnk8s.io/setting-cpu-memory-limits-requests](https://learnk8s.io/setting-cpu-memory-limits-requests)
-   [https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b)
-   [https://nodramadevops.com/2019/10/docker-cpu-resource-limits/](https://nodramadevops.com/2019/10/docker-cpu-resource-limits/)

And finally, if you've enjoyed this thread, you might also like:

-   The Kubernetes workshops that we run at Learnk8s [https://learnk8s.io/training](https://learnk8s.io/training)
-   This collection of past threads [https://twitter.com/danielepolencic/status/1298543151901155330](https://twitter.com/danielepolencic/status/1298543151901155330)
-   The Kubernetes newsletter I publish every week [https://learnk8s.io/learn-kubernetes-weekly](https://learnk8s.io/learn-kubernetes-weekly)