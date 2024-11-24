---
title: "Avoiding Common Pitfalls: Mastering Readiness and Liveness Probes in Kubernetes."
source: https://www.kubernomics.com/post/avoiding-common-pitfalls-mastering-readiness-and-liveness-probes-in-kubernetes
clipped: 2024-02-05
published: 
category: k8s
tags:
  - pod
read: true
---

Reliability is of utmost importance to most applications and Kubernetes provides two key tools to help you manage it: readiness and liveness probes. Yet, counterintuitively, overusing these capabilities can actually lead to less reliability, not more. As such, its important to understand what these capabilities do, when to use which, and how to get the most out of them. Lets start by briefly reviewing what these probes do.

## Readiness Probes

Many Kubernetes workloads are consumed by other resources, inside or outside your cluster. The resources are exposed by "load balancers". These load balancers can be Kubernetes Services or Kubernetes Ingresses. The load balancers rely on the EndpointSlice api, to know which containers are ready and which aren't. Kubelet uses the readiness probe to maintain the EndpointSlice api. A basic readinessProbe config may look like the following:

![[Raw/Media/Resources/df68eba9c8e0512636e732f06a391ae8_MD5.png]]

Readiness probe configuration includes (1) the mechanism to check for readiness, (2) how frequently to execute the check, (3) how many consecutive checks define readiness.

There are several options for the mechanism, including [http](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#http-probes) and [tcp](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-tcp-liveness-probe) calls along with shell commands. In the example above, we are telling the load balancer to do an http get to /healthz on port 80 and considering a 200 http response as successful. periodSeconds specifies the frequency of checks, which dictates how quickly load balancers will be aware of a change in state. The default is 10 seconds, but we prefer a lower value, especially for [zero downtime deployments](https://www.kubernomics.com/post/achieving-zero-downtime-deployments-understanding-pod-startup-and-rolling-updates). Finally, failureThreshold dictates the number of consecutive failures before considering the container not ready. There is a successThreshold which is the same, but for successes. There are additional configuration [options](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#configure-probes) available as well.

Based on your readiness probe configuration, load balancers perform readiness checks on your container. Based on the readiness state, the container is either added to or removed from the load balancer.

## Liveness Probes

Liveness probes are a mechanism by which Kubernetes can decide if your container is still alive and working. Unlike readiness probes, liveness checks are performed by kubelet. If the liveness probe fails for failureThreshold consecutive checks, the container is killed by the kubelet.

Liveness probes have similar configuration to readiness probe, so for the sake of brevity, we won't cover it here.

## What to check for inside your Readiness and Liveness Probes?

Now that we have the basics out of the way, lets discuss what you should actually check inside your probes. Your intuition may say this is a simple question, you should check the health of all the resources required by your workload - your database, any services consumed, etc.

There isn't a one size fits all answer, but your intuition may be misleading you. Consider a situation where your application is consuming another service and that service experiences an intermittent issue. If your probe depends on the availability of the dependent service, you may end up in a situation where all of your service replicas are not ready or worse yet being killed/restarted by kubelet (in the case of a liveness probe), potentially exacerbating the issue.

So what should you do instead?

1.  Check local resources that are isolated to your container/pod, such as whether your http server is running (using an http probe automatically checks this), whether a local file or in process database is available or whether your cache has been warmed.
    
2.  Aggregate failure statistics on any remote resources, such as databases and dependent services and perform checks against these aggregate statistics. A simple way of doing this is capturing in memory the number of calls and failures to these in a running statistics aggregation. Your probe can then consider this aggregate rate vs a set threshold when considering success and failure. Doing this reduces the risk of intermittent or partial dependency failures causing outages for your service.
    

This consideration becomes even more important for liveness checks as the time to kill and restart a container could be non-deterministic, especially if you have configured your deployment to re-pull container images.

## When should you use Readiness and Liveness probes?

Even with the potential pitfalls outlined above, readiness and liveness probes are your friends!

Your serving workloads should always include a readiness probe. This probe is critical to ensure that any required caches are warmed on startup and incoming requests are handled until the container is shut down on termination. They are especially important for [zero downtime deployments](https://www.kubernomics.com/post/achieving-zero-downtime-deployments-understanding-pod-startup-and-rolling-updates).

Liveness probes should be used for long running jobs or poll based workloads (such as kafka consumers) that may be subject to deadlock. For these workloads, consider tracking whether the workload is making "forward progress" and using this information when responding to a liveness probe check.

Consider avoiding liveness probes for serving workloads or limiting them local resource availability to reduce the risk of a distributed outage.

What has been your experience with readiness and liveness probes? Have they worked well for you? Have you experience any unexpected side effects? Reply and let us know!