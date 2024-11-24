---
title: Sticky sessions and canary releases in Kubernetes
source: https://dev.to/danielepolencic/sticky-sessions-and-canary-releases-in-kubernetes-5a92
clipped: 2023-09-08
published: 2023-06-19
category: network
tags:
  - network
  - cni
read: false
---

Sticky sessions or session affinity is a convenient strategy to keep subsequent requests always reaching the same pod.

Let's look at how it works by deploying a sample application with three replicas and one service.

In this scenario, **requests directed to the service are load-balanced amongst the available replicas.**

[![[Raw/Media/Resources/7d30ef14c55de3c755e616b00940493a_MD5.png]]](https://res.cloudinary.com/practicaldev/image/fetch/s--4-n_g6I3--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://res.cloudinary.com/learnk8s/image/upload/v1687158034/threads/sticky-2_m4unrn.png)

Let's deploy the [ingress-nginx controller](https://github.com/kubernetes/ingress-nginx) and create an Ingress manifest for the deployment.

In this case, **the ingress controller skips the services** and load balances the traffic directly to the pods.

[![[Raw/Media/Resources/361574f378207b1beb0d4f81018ab792_MD5.png]]](https://res.cloudinary.com/practicaldev/image/fetch/s--XQIYJlyw--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://res.cloudinary.com/learnk8s/image/upload/v1687158033/threads/sticky-3_bubeg0.png)

While the two scenarios end up with the same outcome (i.e. requests are distributed to all replicas), there's a subtle (but essential) distinction: **the Service operates on L4 (TCP/UDP), whereas the Ingress is L7 (HTTP).**

[![[Raw/Media/Resources/c9a142b41f36100a3013806cece18a5f_MD5.png]]](https://res.cloudinary.com/practicaldev/image/fetch/s--iZkENQA7--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://res.cloudinary.com/learnk8s/image/upload/v1687158033/threads/sticky-4_pjzvvg.png)

**Unlike the service, the Ingress controller can route traffic based on paths, headers, etc.**

You can also use it to define weights (e.g. 20-80 traffic split) or sticky sessions (all requests from the same origin always land on the same pod).

[![[Raw/Media/Resources/1420c71be89890fe2e4081a7c3f534e0_MD5.png]]](https://res.cloudinary.com/practicaldev/image/fetch/s--VX_qjS7d--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://res.cloudinary.com/learnk8s/image/upload/v1687158034/threads/sticky-5_p4yqf4.png)

The following Ingress implements **sticky sessions for the nginx-ingress controller.**

The ingress writes a cookie on your browser to keep track of what instance you visited.

[![[Raw/Media/Resources/5509f544bc6b7f38ab6cddf88f9728fd_MD5.png]]](https://res.cloudinary.com/practicaldev/image/fetch/s--mady2YrK--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://res.cloudinary.com/learnk8s/image/upload/v1687158034/threads/sticky-6_sbpwle.png)

There are two convenient settings for affinity:

1.  **balanced** — requests are redistributed if the deployment scales up.
2.  **persistent** — no matter what, the requests always stick to the same pod.

**nginx-ingress can also be used for canary releases.**

If you have two deployments and you wish to test a subset of the traffic for a newer version of that deployment, you can do so with a canary release (and impact a minimal amount of users).

[![[Raw/Media/Resources/5a6d4f426cf254550f8a5a91b3c2f6a2_MD5.png]]](https://res.cloudinary.com/practicaldev/image/fetch/s--7gp67UPC--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://res.cloudinary.com/learnk8s/image/upload/v1687158034/threads/sticky-8_ebkju9.png)

**In a canary release, each deployment has its own Ingress manifest.**

However, one of those is labelled as a canary.

You can decide how the traffic is forwarded: for example, you could inspect a header or cookie.

[![[Raw/Media/Resources/f062113b6f068bae48674c132dcd4d2e_MD5.png]]](https://res.cloudinary.com/practicaldev/image/fetch/s--59m6fkXD--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://res.cloudinary.com/learnk8s/image/upload/v1687158034/threads/sticky-9_s5g811.png)

In this example, all traffic labelled east-us is routed to the canary deployment.

[![[Raw/Media/Resources/d15a8359333316028ca5810650fa0dde_MD5.png]]](https://res.cloudinary.com/practicaldev/image/fetch/s--fwzsL2Z2--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://res.cloudinary.com/learnk8s/image/upload/v1687158035/threads/sticky-10_c3nu9n.png)

You can also decide which fraction of the total traffic is routed to the canary with weights.

[![[Raw/Media/Resources/92530d9e8a458da1d962daca6b63c344_MD5.png]]](https://res.cloudinary.com/practicaldev/image/fetch/s--3gCAvmJQ--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://res.cloudinary.com/learnk8s/image/upload/v1687158034/threads/sticky-11_ymwlj7.png)

But if the header is omitted in a subsequent request, the user will return to see the previous deployment.

*How can you fix that?*

[![[Raw/Media/Resources/7428d16a76ebcf5bef41967a17fb8805_MD5.png]]](https://res.cloudinary.com/practicaldev/image/fetch/s--RlvAyQMl--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://res.cloudinary.com/learnk8s/image/upload/v1687158034/threads/sticky-12_llofbe.png)

*With sticky sessions!*

**You can combine canary releases and sticky sessions with ingress-nginx to progressively (and safely) roll out new deployments to your users.**

[![[Raw/Media/Resources/0e43b9e4a47646cbfeb4edfce68fcb94_MD5.png]]](https://res.cloudinary.com/practicaldev/image/fetch/s--p9fIfrNA--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://res.cloudinary.com/learnk8s/image/upload/v1687158035/threads/sticky-13_vvrcgl.png)

It's important to remember that those types of canary releases are only possible for front-facing apps.

To roll out a canary release for internal microservices, you should look at alternatives (e.g. service mesh).

[![[Raw/Media/Resources/e4dc93473a2fc1f531a3c21d90954056_MD5.png]]](https://res.cloudinary.com/practicaldev/image/fetch/s--whdFtjfL--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://res.cloudinary.com/learnk8s/image/upload/v1687158035/threads/sticky-14_bfkjit.png)

*Is nginx-ingress the only option for sticky sessions and canary releases?*

Not really, but the annotations might be different to other ingress controllers.

[At Learnk8s, we've put together a spreadsheet to compare them.](https://docs.google.com/spreadsheets/d/191WWNpjJ2za6-nbG4ZoUMXMpUK8KlCIosvQB0f-oq3k)

And finally, if you've enjoyed this thread, you might also like:

-   [The Kubernetes workshops that we run at Learnk8s.](https://learnk8s.io/training)
-   [This collection of past threads.](https://twitter.com/danielepolencic/status/1298543151901155330)
-   [The Kubernetes newsletter I publish every week.](https://learnk8s.io/learn-kubernetes-weekly)

While authoring this post, I also found the following resources valuable:

-   [Configure a canary deployment](https://docs.mirantis.com/mke/3.6/ops/deploy-apps-k8s/nginx-ingress/configure-canary-deployment.html)
-   [Sticky sessions/Session Affinity based on Nginx Ingress](https://help.ovhcloud.com/csm/en-sg-public-cloud-kubernetes-sticky-session-nginx-ingress?id=kb_article_view&sysparm_article=KB0049968)
-   [Session Affinity and Kubernetes— Proceed With Caution!](https://pauldally.medium.com/session-affinity-and-kubernetes-proceed-with-caution-8e66fd5deb05)
-   [Session Affinity](https://github.com/kubernetes/ingress-nginx/blob/main/docs/user-guide/nginx-configuration/annotations.md#session-affinity)