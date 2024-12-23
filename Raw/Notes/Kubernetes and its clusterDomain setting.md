---
title: Kubernetes and its clusterDomain setting
source: https://stackable.tech/en/kubernetes-clusterdomain-setting/
clipped: 2024-12-23
published: 2024-10-31
category: k8s
tags:
  - network
read: false
---

on 31\. October 2024 | Featured

## **TL;DR**

DNS resolution in Kubernetes clusters is a highly complex topic, and not without potential ambiguities. Depending on namespaces, conflicts with global top-level domains (gTLDs) can occur that lead to inconsistent resolution depending on where an application runs.

In this post we will explain the stumbling stones that exist here, as well as how Kubernetes tries to solve these with the `clusterDomain` setting. We also show, why it is not currently reliably possible to determine the effective `clusterDomain` for any given cluster and explore possible ways for improving this situation.

## Introduction

This might seem like a niche problem, but it could save you a lot of headaches if you ever run into it. Let’s dive into a specific issue of DNS resolution in Kubernetes: the cluster domain.

Kubernetes assigns DNS names to services (for Pods it’s [more complicated](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#a-aaaa-records-1)) based on their name and the namespace they belong to. This naming system is essential for enabling services to discover each other easily.

Here’s a simple example to illustrate this:

```
# https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/
$ kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
pod/dnsutils created

$ kubectl create namespace foo
namespace/foo created

$ kubectl --namespace foo create service clusterip foo-service --tcp=80
service/foo-service created

$ kubectl exec -i -t dnsutils -- dig +short +search foo-service.foo
10.96.84.177

$ kubectl create namespace bar
namespace/bar created

$ kubectl --namespace bar create service clusterip bar-service --tcp=80
service/bar-service created

$ kubectl exec -i -t dnsutils -- dig +short +search bar-service.bar
10.96.115.165
```

Everything seems to work well, right? Services can communicate by knowing just their names and namespaces.

Right?

Right?

Let’s create another namespace that might lead to an issue:

```
// Oh oh... `app` is a gTLD!
$ kubectl create namespace app
namespace/app created

$ kubectl --namespace app create service clusterip get --tcp=80
service/get created

// In Kubernetes:
$ kubectl exec -i -t dnsutils -- dig +short +search get.app
10.96.251.201
```

But what if you try to resolve it outside of Kubernetes?

```
// Now try outside - https://get.app :
$ dig +short +search get.app
216.239.32.27
```

Uh-oh. It turns out “app” is a gTLD (global top-level domain). Depending on where the query originates, you might end up resolving to an external IP instead of the expected service. This creates an ambiguity issue that could lead to unreliable service resolution.

**The Solution:** **clusterDomain** **to the Rescue!**

To avoid ambiguity, Kubernetes provides the clusterDomain setting which is used to build DNS names that are fully qualified and unique within the cluster.

Let’s assume, for now, this setting is set to `cluster.local` (which is not actually the default).

For example, with clusterDomain set to `cluster.local`, our service `foo-service` in the foo namespace would have the FQDN: `foo-service.foo.svc.cluster.local`. This setup eliminates conflicts with external DNS names.

## What About Pods?

Pods need to resolve DNS names. How does that work? The answer is that the kubelet does a lot of work behind the scenes with the result that the /etc/resolv.conf file is automagically updated by the kubelet.

This is handled in a method called [GetPodDNS](https://github.com/kubernetes/kubernetes/blob/9d967ff97332a024b8ae5ba89c83c239474f42fd/pkg/kubelet/network/dns/dns.go#L385-L448) which looks at the configured DNSPolicy of a Pod and then configures it accordingly. The [official docs](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy) on this are good. Funnily enough there is a setting called Default which is not actually the default:

![[Raw/Media/Resources/716dd0510e7bd8fd92f28cf6da8557c1_MD5.png]]

Using the default `ClusterFirst` policy the kubelet updates our `/etc/resolv.conf` file to look like this:

```
search stackable-operators.svc.cluster.local svc.cluster.local cluster.local localdomain
nameserver 10.96.0.10
options ndots:5
```

*Aside: Do you see that `ndots` setting? Want to save 80% of DNS lookups in your cluster with one easy step? DNS providers hate this trick: [https://github.com/stackabletech/issues/issues/656](https://github.com/stackabletech/issues/issues/656)*

So, as you can see (if you’ve ever seen a `resolv.conf` file that is), this is the magic that allows those relative lookups without specifying the FQDN. Because it’ll also try appending all the values from the `search` list. BUT this only works if you set the DNS Policy to `ClusterFirst` and our users are free to use [pod overrides](https://docs.stackable.tech/home/stable/concepts/overrides.html) to change the pods our operators deploy.  
One could argue: You’re on your own if you muck around with the internals but it’d be nice to cover that use case as well.

On top of this: It is implementation dependent whether that `search` list is taken into account and I wouldn’t be surprised if there are other surprises out there. `dig` for example ignores the `search` unless you specify `+search`.

But, at least we now understand how pods do their DNS resolving. Kinda.

## How do we get the clusterDomain value?

Ideally, we want an automated way to discover the configured clusterDomain value, so we don’t need to hardcode it.  
Spoiler: Unfortunately, there’s no straightforward Kubernetes API for this purpose, which means we need to get creative.

Some Kubernetes distributions have created their own ways to store and retrieve the clusterDomain:

-   [**k3s**](https://k3s.io/)**:** Uses a ConfigMap (`kube-system/clusterdns`) to store the value in the clusterDomain field.
    -   [https://github.com/k3s-io/k3s/pull/1785](https://github.com/k3s-io/k3s/pull/1785)
-   [**kubeadm**](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) (including [kind](https://kind.sigs.k8s.io/)): Stores the value in a ConfigMap (kube-system/kubeadm-config) that contains the entire kubeadm configuration, including networking.dnsDomain.
-   [**OpenShift**](https://www.redhat.com/en/technologies/cloud-computing/openshift): Defines a custom DNS CRD that sets `.status.clusterDomain`. This provides an easy way to find the clusterDomain in OpenShift environments.
    -   [https://docs.openshift.com/container-platform/4.17/networking/dns-operator.html#nw-dns-view\_dns-operator](https://docs.openshift.com/container-platform/4.17/networking/dns-operator.html#nw-dns-view_dns-operator)

These methods, however, are inconsistent across distributions, making it hard to handle this issue reliably without implementing custom solutions for each environment. Most distributions and cloud providers default to `cluster.local` but we can’t rely on it and we have actual customers [reporting problems](https://github.com/stackabletech/issues/issues/436) when using a different clusterDomain setting.

## Our Options

### Parsing resolv.conf

One option we have is to parse the `resolv.conf` file inside our pods, which contains the value we’re looking for. While this approach could work, it’s not ideal—it depends on the format and structure of `resolv.conf`, which might vary and is an implementation detail of the kubelet implementation. [Parsing](https://stackoverflow.com/questions/1732348/regex-match-open-tags-except-xhtml-self-contained-tags/1732454#1732454) system configuration files can also introduce inconsistencies and errors, making it less reliable.

Unfortunately, this still seems to be one of the better options out there for now.

### kubelet config

In the end this is a kubelet setting called `clusterDomain` in the [Kubelet configuration](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/) or the `cluster-domain` command line parameter.

Can we maybe get access to the config at runtime? Turns out [we can](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/#viewing-the-kubelet-configuration).

```
# kubectl get --raw /api/v1/nodes/kind-control-plane/proxy/configz | jq -r .kubeletconfig.clusterDomain
cluster.local
```

Phew…we’re good! [Are we though?](https://github.com/kubernetes/kubernetes/blob/9d967ff97332a024b8ae5ba89c83c239474f42fd/staging/src/k8s.io/component-base/configz/OWNERS)

```
# For lack of a better idea, assigned configz to api-approvers because
# configz is an API that has been around for a long time, even if we don't
# guarantee its stability.
```

Welp. Back to square one – we need a stable API. And in addition we aren’t entirely sure if we’ll always have access to this API from within our operators or if we need additional RBAC rules to guarantee access, maybe it can be disabled by distros as well? We didn’t dig into that. If anyone knows the answer, please let us know.

### KEPs to the Rescue?

We reached out to the [Kubernetes community](https://slack.k8s.io/) on the [#sig-network](https://kubernetes.slack.com/archives/C09QYUH5W) Slack channel to [discuss this issue](https://kubernetes.slack.com/archives/C09QYUH5W/p1729521715336479). During these discussions, we came across two related Kubernetes Enhancement Proposals (KEPs):

-   [KEP-4827: Component Statusz](https://github.com/kubernetes/enhancements/tree/master/keps/sig-instrumentation/4827-component-statusz): This KEP aims to add a `statusz` endpoint for core Kubernetes components, enhancing observability and diagnostics. While useful, it isn’t directly related to our need for retrieving the cluster domain value.
-   [KEP-4828: Component Flagz](https://github.com/kubernetes/enhancements/blob/master/keps/sig-instrumentation/4828-component-flagz/README.md): This KEP proposes adding a `flagz` endpoint that exposes the flags used to run Kubernetes components, which could include `cluster-domain`. While this may work, it still lacks a [machine-readable format](https://github.com/kubernetes/enhancements/tree/e1d4725d07a07c40da7ded6e1dd44c0f9aaeaeb9/keps/sig-instrumentation/4828-component-flagz#endpoint-response-format) that would make automation easy and consistent. Currently, it appears to be more geared towards human inspection. According to the discussion this restriction might be lifted later. This is definitely one [KEP](https://github.com/kubernetes/enhancements/issues/4828) to keep in mind!

### Idea: Downward API

While reviewing this very blog post, [Antonio Ojea](https://github.com/aojea) suggested another option to solve this problem: Using an addition to the [Downward API](https://kubernetes.io/docs/concepts/workloads/pods/downward-api/).

This means any container could request the current value of the `clusterDomain` setting to be *injected* into the container itself. In a pod specification, Antonio suggested this could look like this:

```
# This would make a variable called MY_CLUSTER_DOMAIN available that contains the clusterDomain
env:
  - name: MY_CLUSTER_DOMAIN
    valueFrom:
      nodePropertyRef: clusterDomain
```

[Tim Hockin](http://hockin.org/~thockin/resume/) suggested this slight alteration:

```
# Option 1: EnvVarSource
# The value ends up in an environment variable called `MY_CLUSTER_DOMAIN`

env:
  - name: MY_CLUSTER_DOMAIN
    valueFrom:
      runtimeConfigs: clusterDomain

# Option 2: EnvVarFrom
# This would probably need to be refined but would create environment variables with names predefined by Kubernetes containing various runtime configuration options.
envFrom:
  - runtimeConfigs

# Option 3
# This would mount a file called "clusterdomain" in /etc/runtimeconfig/clusterdomain
# Containing a line with the clusterDomain in it
volumeMounts:
  - name: runtimeConfig
    mountPath: /etc/runtimeconfig

volumes:
  - name: runtimeConfig
    downwardAPI:
      items:
        - path: "clusterdomain"
          runtimeConfigs:
            - clusterDomain
```

We think this solution is elegant and simple (from the user’s perspective) and solves all the existing issues related to discovering the cluster domain. It’s the best proposal we’ve seen so far but will probably require a discussion about the exact naming of things.

### Next Steps

We’ve been invited to present our case to the SIG Network team in one of their biweekly meetings.

This article was actually written in preparation for that meeting so we’re, well, well prepared 🙂

Our hope is to expose a stable API to get this value and the Downward API is our best suggestion so far. The fact that k3s, kind and OpenShift invented something themselves kinda shows the need for it.  

During [said meeting](https://docs.google.com/document/d/1_w77-zG_Xj0zYvEMfQZTQ-wPP4kXkpGD8smVtW_qqWM/edit?tab=t.0#heading=h.537niz7s50fv) the consensus was that this is a good idea and that a KEP is needed for the next steps as a few other SIGs will want to chime in. So, our very own [Natalie](https://github.com/nightkr/) offered to drive this forward and create a first draft.

## The (Temporary) Solution

For now, we’ve decided to implement a temporary workaround by allowing users to manually set an environment variable on our operators to define their clusterDomain. While this solution isn’t as elegant as automatic discovery, it provides a straightforward way for users to ensure the correct configuration.

Here’s how it works: Users can define their clusterDomain value as an environment variable when deploying our operators. This allows for explicit control, ensuring that the right domain is used even in custom Kubernetes environments where `cluster.local` might not be the default. For more information please refer to [https://docs.stackable.tech/home/nightly/guides/kubernetes-cluster-domain](https://docs.stackable.tech/home/nightly/guides/kubernetes-cluster-domain).

However, there are two main drawbacks:

-   Operator Lifecycle Manager (OLM) on OpenShift presents some limitations. It doesn’t easily allow passing environment variables for all of our operators, which complicates the deployment process in certain cases.
-   Manual Effort: Users need to remember to configure this value each time they deploy the operator. If forgotten, it could lead to incorrect or unpredictable behavior in the cluster.

We recognize that manual configuration is less than ideal, and we’re aiming to replace this solution with a more automated approach in the future.

## Summary & Thank You

As you can see, what seemed like a simple configuration question about Kubernetes’ `clusterDomain` can spiral into unexpected complexity. Our journey through resolving this issue involved discovering workarounds, diving into Kubernetes Enhancement Proposals, and even discussing the matter with the Kubernetes community.

We want to give credit to our colleagues – Malte, Natalie, Nick, Sascha, and Sebastian – who put in the effort to research and understand the intricacies of this problem. Also, a big thank you to our customer [Discovery](https://www.discovery.co.za/portal/) for sponsoring part of this research. 

And thank you to Antonio Ojea, Tim Hockin and others from the Kubernetes community for jumping on this thread and discussing our use-case.