---
title: What happens when you create a pod in Kubernetes
source: https://itnext.io/what-happens-when-you-create-a-pod-in-kubernetes-6b789b6db8a8
clipped: 2023-09-04
published: 
category: k8s
tags:
  - k8s
  - pod
read: true
---

What happens when you create a Pod in Kubernetes?

A surprisingly simple task reveals a complicated workflow that touches several components in the cluster.

Let’s start with the obvious: **kubectl sends the YAML definition to the API server.**

In this step, kubectl:

-   Discovers the API endpoints using OpenAPI (Swagger).
-   Negotiates the resource version.
-   Validates the YAML.
-   Issues the request.

When the request reaches the API, it goes through the following:

-   **Authentication & authorization.**
-   **Admission controllers.**

In the last step, it’s finally stored in etcd.

After this, the pod is added to the scheduler queue.

**The scheduler filters and scores the nodes to find the best one.**

And it finally binds the pod to the node.

The binding is written in etcd.

At this point, the pod exists only in etcd as a record.

**The infrastructure hasn’t created any containers yet.**

Here’s where the kubelet takes over.

The kubelet pulls the Pod definition and proceeds to delegate:

1.  Network creation to the CNI (e.g. Cilium).
2.  Container creation to the CRI (e.g. containerd).
3.  Storage creation to the CSI (e.g. OpenEBS).

Among other things, the Kubelet will execute the Pod’s probes and, when the Pod is running, report its IP address to the control plane.

**That IP and the containers’ ports are stored as endpoints in etcd.**

*Wait… endpoint what?*

In Kubernetes:

-   endpoint is a 10.0.0.2:3000 (IP:port) pair.
-   Endpoint is a collection of endpoints (a list of IP:port pairs).

For every Service in the cluster, **Kubernetes creates an Endpoint object with endpoints.**

*Confusing, isn’t it?*

The endpoints (IP:port) are used by:

-   kube-proxy to set iptables rules.
-   CoreDNS to update the DNS entries.
-   Ingress controllers to set up downstreams.
-   Service meshes.
-   And more operators.

As soon as an endpoint is added, the components are notified.

When the endpoint (IP:port) is propagated, you can finally start using the Pod!

*What happens when you delete a Pod?*

The exact process but in reverse.

This is annoying because there are few opportunities for race conditions.

The correct sequence is:

1.  App stops accepting connections.
2.  Controllers (kube-proxy, ingress, etc.) to remove the endpoint.
3.  App to drain existing connection.
4.  App to shut down.

If you want to learn more about the graceful shutdown in Kubernetes, you can find my article here [https://learnk8s.io/graceful-shutdown](https://learnk8s.io/graceful-shutdown)

And finally, if you’ve enjoyed this thread, you might also like the Kubernetes workshops that we run at Learnk8s [https://learnk8s.io/training](https://learnk8s.io/training) or this collection of past Twitter threads [https://twitter.com/danielepolencic/status/1298543151901155330](https://twitter.com/danielepolencic/status/1298543151901155330)