---
title: Cilium for Kubernetes networking Why we use it and why we love it
source: https://blog.palark.com/why-cilium-for-kubernetes-networking/
clipped: 2023-09-04
published: 
category: network
tags:
  - k8s
  - network
  - cilium
read: false
---

Thanks to the CNI (Container Network Interface), Kubernetes offers a good deal of options to address your networking needs. After years of relying on a simple solution, we faced a growing demand for advanced features backed by our customers’ needs. Cilium brought the networking in our K8s platform to the next level.

## Background

We build and maintain infrastructure for companies varying in industry, size, and technology stack. Their applications are deployed to private and public clouds as well as bare-metal servers. They have different requirements for fault tolerance, scalability, financial expenses, security, and so forth. When offering [our services](https://www.palark.com/services), we need to meet all those expectations while being efficient enough to cope with emerging infrastructure-related diversity.

When we built our early Kubernetes-based platform years ago, we set out to achieve a production-ready, simple, reliable solution based on staunch Open Source components. To achieve that, the natural choice of our CNI plugin seemed to be  [Flannel](https://github.com/flannel-io/flannel) (with kube-proxy as its companion).

The most popular options available at that time were Flannel and Weave Net. Flannel was more mature, had minimal dependencies, and was easy to install. Our benchmarks also proved it performed at a high level. Consequently, we went for it and ended up happy with our choice.

At the same time, we had no doubt we would hit its limits one day.

## Following the growing demand

As time went on, we got more customers, more Kubernetes clusters, and more specific requirements for the platform. We encountered an increasing need for better security, performance, and observability. Those needs applied to various infrastructure elements, and networking was obviously one of them. Eventually, we realised it was time to move on to a more advanced CNI plugin.

Numerous issues propelled us to make the leap to the next chapter:

1.  One financial organisation had a strict “everything is forbidden by default” rule enforced.
2.  The cluster of one widely used web portal had a vast number of services which had an overwhelming effect on kube-proxy.
3.  PCI DSS compliance required another customer to implement flexible and powerful network policies management with decent observability on top of it.
4.  Multiple other applications experiencing huge incoming traffic faced performance issues in iptables & netfilter, which Flannel uses.

We couldn’t be held back by the existing limitations any longer and decided to find another CNI to use in our Kubernetes platform — one that could handle all the new challenges.

## Why Cilium?

There are lots of [CNI options](https://www.cni.dev/docs/#3rd-party-plugins) available today. We wanted to stick to [eBPF](https://en.wikipedia.org/wiki/Berkeley_Packet_Filter), which proved to be a powerful technology providing many benefits in terms of observability, security, etc. With that in mind, two well-known projects arise when you think of CNI plugins: [Cilium](https://cilium.io/) and [Calico](https://www.tigera.io/project-calico/).

On the whole, both of them are pretty great. However, we still needed to choose one of them. Cilium seemed more widely used and discussed in the community: better GitHub stats (such as stars, forks, and contributors) can be used as a certain argument to prove its worth. It is also a CNCF project. While it doesn’t guarantee too much, that is still a valid point, all things being equal.

After reading various articles on Cilium, we decided to try it out and performed various tests on a few different K8s clusters. It turned out to be a purely positive experience that revealed even more features and benefits than we had expected.

## The main Cilium features we enjoyed

Here is a list of the things we liked about Cilium in considering whether to use it to address the abovementioned issues we were having:

### 1\. Performance

Using bpfilter (instead of iptables) for routing [means](https://www.admin-magazine.com/Archive/2019/50/Bpfilter-offers-a-new-approach-to-packet-filtering-in-Linux) shifting filtering tasks to the kernel space, which yields impressive gains in performance. This is just what was promised by the project’s design, a great many articles, and 3rd-party benchmarks. Our own tests confirmed a significant improvement in the processing traffic speed compared to Flannel + kube-proxy, which we used before.

[![[Raw/Media/Resources/1d928d5a595d26add7efe2a6c151343d_MD5.png]]](https://blog.palark.com/wp-content/uploads/2023/03/ebpf-host-routing-diagram.png)

*eBPF host-routing compared to using iptables. Source: [“CNI Benchmark: Understanding Cilium Network Performance”](https://cilium.io/blog/2021/05/11/cni-benchmark/) (Cilium blog)*

Helpful resources on this subject include:

1.  [“Why is the kernel community replacing iptables with BPF?”](https://cilium.io/blog/2018/04/17/why-is-the-kernel-community-replacing-iptables/) (Cilium blog);
2.  [“BPF, eBPF, XDP and Bpfilter… What are These Things and What do They Mean for the Enterprise?”](https://www.netronome.com/blog/bpf-ebpf-xdp-and-bpfilter-what-are-these-things-and-what-do-they-mean-enterprise/) (Netronome);
3.  [“kube-proxy Hybrid Modes”](https://docs.cilium.io/en/v1.9/gettingstarted/kubeproxy-free/#kube-proxy-hybrid-modes) (Cilium docs).

### 2\. Better network policies

[CiliumNetworkPolicy CRD](https://docs.cilium.io/en/latest/network/kubernetes/policy/#ciliumnetworkpolicy) extends the Kubernetes NetworkPolicy API. It brings such features as L7 (instead of just L3/L4) network policies support for ingress & egress and [port ranges](https://github.com/cilium/cilium/issues/16622) specification in network policies.

As Cilium developers note: “Ideally all of the functionality will be merged into the standard resource format and this CRD will no longer be required.”

### 3\. Inter-node traffic control

You can control inter-node traffic thanks to the [CiliumClusterwideNetworkPolicy](https://docs.cilium.io/en/latest/network/kubernetes/policy/#ciliumclusterwidenetworkpolicy). These policies are applicable to the whole cluster (non-namespaced) and provide you with the means to specify nodes as the source and target. It renders filtering traffic between different node groups convenient.

### 4\. Policy enforcement modes

Life is made even easier with its easy-to-use [policy enforcement modes](https://docs.cilium.io/en/latest/security/policy/intro/#policy-enforcement-modes). The *default* mode is a good fit in most cases: with no initial restrictions, but as soon as something is allowed, all the rest becomes restricted. *Always* mode — when policy enforcement is on for all endpoints — is helpful for environments with higher security requirements.

### 5\. Hubble and its UI

[Hubble](https://github.com/cilium/hubble) is a truly fantastic tool for network & service observability as well as visual rendering. Specifically, it monitors the traffic and updates the services interaction diagram in real time. You can easily see which requests are in the works, the related IPs, how network policies are applied, etc.

Now for a couple of examples of how Hubble is used in my Kubernetes sandbox. First of all, here we have the namespace with the Ingress-NGINX controller. We can see that an external user enters the Hubble UI after being authorised by Dex. Who could it be?..

![[Raw/Media/Resources/8bc0a2379126d73ff6f4b3b616f4fadf_MD5.png]]

Now, here’s a little more interesting example: Hubble spent about a minute visualising how the Prometheus namespace communicates with the rest of the cluster. You can see that Prometheus has scraped metrics from numerous services. What a great feature! You should’ve known about it before you spent hours drawing all those infrastructure diagrams for your projects!

![[Raw/Media/Resources/c3e1301031b981f3816071556f01590f_MD5.png]]

### 6\. Visual policy editor

[This online service](https://editor.cilium.io/) provides an easy-to-use, mouse-friendly UI to create the rules and get the corresponding YAML configuration to apply them. The only missing thing I have to gripe about here is the lack of an opportunity to make a reverse visualisation of existing configurations.

Once again, this list is far from a complete set of Cilium features. It’s just a biased selection of them I’ve made based on what we needed and what we’re most excited about.

## What Cilium has done for us

Let’s take a look back at the specific issues our customers were having that propelled our interest in getting started using Cilium in the Kubernetes platform.

The “everything is forbidden by default” rule in the first case was implemented using the abovementioned policy enforcement modes. Generally, we would rely on *default* mode by specifying a full list of what is allowed in this specific environment and forbidding everything else.

Here are a few examples of rather simple policies which may prove helpful for others. You will most likely have dozens or hundreds of them.

1.  Allow any Pod to access Istio endpoints:

```
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: all-pods-to-istio-internal-access
spec:
  egress:
  - toEndpoints:
    - matchLabels:
        k8s:io.kubernetes.pod.namespace: infra-istio
    toPorts:
    - ports:
      - port: "8443"
        protocol: TCP
  endpointSelector: {}
```

2.  Allow all traffic inside a given namespace:

```
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-ingress-egress-within-namespace
spec:
  egress:
  - toEndpoints:
    - {}
  endpointSelector: {}
  ingress:
  - fromEndpoints:
    - {}
```

3.  Allow VictoriaMetrics to scrape all Pods in a given namespace:

```
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: vmagent-allow-desired-namespace
spec:
  egress:
  - toEndpoints:
    - matchLabels:
        k8s:io.kubernetes.pod.namespace: desired-namespace
  endpointSelector:
    matchLabels:
      k8s:io.cilium.k8s.policy.serviceaccount: victoria-metrics-agent-usr
      k8s:io.kubernetes.pod.namespace: vmagent-system
```

4.  Allow the Kubernetes Metrics Server to reach the kubelet port:

```
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: host-firewall-allow-metrics-server-to-kubelet
spec:
  ingress:
  - fromEndpoints:
    - matchLabels:
        k8s:io.cilium.k8s.policy.serviceaccount: metrics-server
        k8s:io.kubernetes.pod.namespace: my-metrics-namespace
    toPorts:
    - ports:
      - port: "10250"
        protocol: TCP
  nodeSelector:
    matchLabels: {}
```

As for other issues, we initially encountered challenges in:

-   Cases #2 and #4, due to the poor performance of the iptables-based networking stack. The benchmarks we mentioned and the tests we performed proved themselves during the actual operations.
-   Hubble provides a sufficient level of observability, which was required in case #3.

## What’s next?

To sum up this experience, we successfully tackled all our pain points related to Kubernetes networking.

What can we say regarding the future of Cilium in general? While it is currently [an incubating CNCF project](https://www.cncf.io/projects/cilium/), it [applied](https://cilium.io/blog/2022/10/27/cilium-applies-for-graduation/) for graduation at the end of last year. It will take some time to accomplish, but this project is headed in a crystal clear direction. Most recently, in February 2023, it [was announced](https://www.cncf.io/blog/2023/02/13/a-well-secured-project-cilium-security-audits-2022-published/) that Cilium passed two security audits, which is an essential step for its further graduation.

We are watching the project’s [roadmap](https://docs.cilium.io/en/latest/community/roadmap/) and waiting for some features and related tools to be implemented or become sufficiently mature. (Yes, [Tetragon](https://github.com/cilium/tetragon), no doubt you’re going to be great!)

For example, while we leverage Kubernetes [EndpointSlice CRD](https://docs.cilium.io/en/latest/network/kubernetes/ciliumendpointslice/) in clusters with high traffic, the relevant [Cilium feature](https://docs.cilium.io/en/latest/network/kubernetes/ciliumendpointslice_beta/) is currently in beta — thus, we’re waiting for its stable release. Another beta feature that we are waiting to become stable is the [Local Redirect Policy](https://docs.cilium.io/en/latest/network/kubernetes/local-redirect-policy/#local-redirect-policy-beta), which redirects Pod traffic locally to another backend Pod within a node instead of a random one within the whole cluster.

## Afterword

After settling on our new networking infrastructure in production environments and evaluating its performance and new features, we are pleased with our decision to adopt Cilium as its benefits are evident. Perhaps it’s not a silver bullet for a diverse and constantly changing cloud-native world, and by no means is it the easiest technology to start working with. Nevertheless, if you have the motivation, knowledge, and a bit of a lust for adventure — it’s 100% worth trying and would most likely pay off manifold.