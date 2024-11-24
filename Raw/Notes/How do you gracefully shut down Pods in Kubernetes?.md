---
title: How do you gracefully shut down Pods in Kubernetes?
source: https://itnext.io/how-do-you-gracefully-shut-down-pods-in-kubernetes-fb19f617cd67
clipped: 2023-09-04
published: 
category: k8s
tags:
  - k8s
  - pod
read: true
---

![[Raw/Media/Resources/05e087f904e7f496577abc9a28e075ba_MD5.png]]

When you type `kubectl delete pod`, the pod is deleted, and the endpoint controller removes its IP address and port (endpoint) from the Services and etcd.

You can observe this with `kubectl describe service`.

![[Raw/Media/Resources/9134a3f5585c60df47fd8b63c72e19e2_MD5.png]]

But that’s not enough!

**Several components sync a local list of endpoints:**

-   kube-proxy keeps a local list of endpoints to write iptables rules.
-   CoreDNS uses the endpoint to reconfigure the DNS entries.

And the same is true for the Ingress controller, Istio, etc.

![[Raw/Media/Resources/5275a3db298e43d548bb0fc8032a3f96_MD5.png]]

All those components will (eventually) remove the previous endpoint so that no traffic can ever reach it again.

At the same time, the kubelet is also notified of the change and deletes the pod.

*What happens when the kubelet deletes the pod before the rest of the components?*

![[Raw/Media/Resources/a8b284fc3cbdae128b1a534a92a45eab_MD5.png]]

**Unfortunately, you will experience downtime** because components such as kube-proxy, CoreDNS, the ingress controller, etc., still use that IP address to route traffic.

*So what can you do?*

Wait!

![[Raw/Media/Resources/b338613c2d3957657809d31f7d9a76e8_MD5.png]]

**If you wait long enough before deleting the pod, the in-flight traffic can still resolve, and the new traffic can be assigned to other pods.**

*How are you supposed to wait?*

![[Raw/Media/Resources/7ca3962f684195035c1dee9bf6c5fe86_MD5.png]]

When the kubelet deletes a pod, it goes through the following steps:

-   Triggers the `preStop` hook (if any).
-   Sends the SIGTERM.
-   Sends the SIGKILL signal (after 30 seconds).

![[Raw/Media/Resources/9620e9e55a54429890be8323173f455d_MD5.png]]

**You can use the** `**preStop**` **hook to insert an artificial delay.**

![[Raw/Media/Resources/f541797c6621fae6355aca6b7aefc449_MD5.png]]

**You can listen to the SIGTERM signal in your app and wait.**

Also, you can gracefully stop the process and exit when you are done waiting.

Kubernetes gives you 30s to do so (configurable).

![[Raw/Media/Resources/449d77e941bab1c4e064a015619f6ae4_MD5.png]]

*Should you wait 10 seconds, 20 or 30s?*

There’s no single answer.

While propagating endpoints could only take a few seconds, Kubernetes doesn’t guarantee any timing nor that all of the components will complete it at the same time.

![[Raw/Media/Resources/7a9b88e3cba9376963d54fe1a010beb9_MD5.png]]

If you want to explore more, here are a few links:

-   [https://learnk8s.io/graceful-shutdown](https://learnk8s.io/graceful-shutdown)
-   [https://freecontent.manning.com/handling-client-requests-properly-with-kubernetes/](https://freecontent.manning.com/handling-client-requests-properly-with-kubernetes/)
-   [https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods)
-   [https://medium.com/tailwinds-navigator/kubernetes-tip-how-to-gracefully-handle-pod-deletion-b28d23644ccc](https://medium.com/tailwinds-navigator/kubernetes-tip-how-to-gracefully-handle-pod-deletion-b28d23644ccc)
-   [https://medium.com/flant-com/kubernetes-graceful-shutdown-nginx-php-fpm-d5ab266963c2](https://medium.com/flant-com/kubernetes-graceful-shutdown-nginx-php-fpm-d5ab266963c2)
-   [https://www.openshift.com/blog/kubernetes-pods-life](https://www.openshift.com/blog/kubernetes-pods-life)

And finally, if you’ve enjoyed this thread, you might also like:

-   The Kubernetes workshops that we run at Learnk8s [https://learnk8s.io/training](https://learnk8s.io/training)
-   This collection of past threads [https://twitter.com/danielepolencic/status/1298543151901155330](https://twitter.com/danielepolencic/status/1298543151901155330)
-   The Kubernetes newsletter I publish every week, “Learn Kubernetes weekly” [https://learnk8s.io/learn-kubernetes-weekly](https://learnk8s.io/learn-kubernetes-weekly)