---
title: Kubernetes’ Controller Manager
source: https://llorllale.github.io/posts/k8s-kube-controller-manager/
clipped: 2023-09-04
published: 
category: k8s
tags:
  - k8s
  - development
read: true
---

[![cover](https://llorllale.github.io/assets/img/Kubernetes-icon-color.svg)](https://llorllale.github.io/assets/img/Kubernetes-icon-color.svg) [Kubernetes](https://kubernetes.io/) is a platform that automates many of the complexities behind deployment, scaling, and management of resources, such as [pods](https://kubernetes.io/docs/concepts/workloads/pods/). Users can configure these resources imperatively using [kubectl](https://kubernetes.io/docs/reference/kubectl/), or declaratively using configuration files (also deployed using kubectl). At the heart of this platform lies a control loop that works to bring the *current state* of those resources to the *desired state*.

[![control loop](https://llorllale.github.io/assets/img/k8s-controller-manager/control%20loop.drawio.svg)](https://llorllale.github.io/assets/img/k8s-controller-manager/control%20loop.drawio.svg)

In this article we will take a brief look at the component that manages the main control loop, [kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/), as well as the [cloud-controller-manager](https://kubernetes.io/docs/concepts/architecture/cloud-controller/) that manages the control loops specific to cloud environments.

[![components](https://llorllale.github.io/assets/img/k8s-controller-manager/components-of-kubernetes.svg)](https://llorllale.github.io/assets/img/k8s-controller-manager/components-of-kubernetes.svg "Image taken from Kubernetes docs. “CM” represents kube-controller-manager and “CCM” represents cloud-controller-manager.") *Image taken from [Kubernetes docs](https://kubernetes.io/docs/concepts/architecture/cloud-controller/#design).  
“CM” represents `kube-controller-manager` and “CCM” represents `cloud-controller-manager`.*

The controller manager is part of Kubernetes’ [control plane](https://kubernetes.io/docs/concepts/overview/components/#control-plane-components)[1](#fn:1) and runs on the master nodes, normally as a standalone pod. It *manages* many built-in controllers for different resources such as deployments or namespaces. You can find the full list of managed controllers [here](https://github.com/kubernetes/kubernetes/blob/95051a63b323081daf8a3fe55a252eb79f0053aa/cmd/kube-controller-manager/app/controllermanager.go#L434-L480).

Kubernetes implements an event-driven architecture with many independent components reacting to events and acting in concert to drive the system to a desired state. The controller manager registers “[watches](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes)” on the [API server](https://kubernetes.io/docs/concepts/overview/components/#kube-apiserver) that open a connection through which events are constantly streamed[2](#fn:2). Each of these events has an associated *action*, such as “add” or “delete”, and a target resource. Here’s an example from the docs:


```
1 2 3 4 5 6 7 8 9 10 11 12 13 14 15  
GET /api/v1/namespaces/test/pods?watch=1&resourceVersion=10245 
--- 
200 OK 
Transfer-Encoding: chunked 
Content-Type: application/json  
{   "type": "ADDED",   "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "10596", ...}, ...} }
{   "type": "MODIFIED",   "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "11020", ...}, ...} 
} ...
```


The event actions are determined by their `type`, and the target resource is the `object`. Note how `object` is just a regular spec (Pod specs in this example).

The controller manager dispatches these events to controllers that have registered themselves to act on them based on the event’s action and the type of the resource. Note that these controllers do *not* realize the end result directly (ie. create containers, create IP addresses, etc.), they merely update resources that are exposed on the API server itself. In other words, they update the *desired state* of those resources by posting the updated spec back to the API server. It is other components, such as the [kubelet](https://kubernetes.io/docs/concepts/overview/components/#kubelet) and the [kube-proxy](https://kubernetes.io/docs/concepts/overview/components/#kube-proxy), perform the actual grunt work derived from the desired state.

[![controller events](https://llorllale.github.io/assets/img/k8s-controller-manager/kube-controller-manager-events.drawio.png)](https://llorllale.github.io/assets/img/k8s-controller-manager/kube-controller-manager-events.drawio.png "High level view of a small subset of what happens when a new deployment is added.") *High level view of a small subset of what happens when a new deployment is added.*

In cloud environments, the [cloud-controller-manager](https://kubernetes.io/docs/concepts/architecture/cloud-controller/) runs alongside the kube-controller-manager. This controller manager operates the same way as the built-in `kube-controller-manager` in principle, with its control loops tailored specifically for management of cloud infrastructure. It is this controller manager that handles updates from the cloud provider: nodes automatically entering or leaving the cluster, provisioning of load balancers, updating IP routes within your cluster, and so on.

Kubernetes is a distributed system that serves as a platform that automates many of the complexities behind deployment, scaling, and management of applications. The `kube-controller-manager` implements many critical control loops that ensure the cluster’s current state matches the user’s desired state. These control loops function independently of each other, listening for events from the API server and modifying resources there as well. Other components, such as the `kube-proxy` and the `kubelet` perform the actual runtime configurations on the cluster’s nodes. In cloud environments, the `cloud-controller-manager` runs alongside the `kube-controller-manager` and implements control loops specific to cloud infrastructure.

---

**Footnotes**