---
title: What are the Differences Between VPA, HPA, CA, and CPA in Kubernetes?
source: https://hwchiu.medium.com/what-are-the-differences-between-vpa-hpa-ca-and-cpa-in-kubernetes-5f74c97001ed
clipped: 2023-11-02
published: 
category: k8s
tags:
  - resources
  - k8s
read: false
---

Kubernetes offers various automatic scaling mechanisms, such as Horizontal Pod Autoscaling (HPA), which dynamically adjusts the number of Pod replicas based on different conditions. This capability enables Pods to handle current traffic efficiently without the constant intervention of administrators to adjust the replica count.

Apart from HPA, Kubernetes also provides other related mechanisms like VPA (Vertical Pod Autoscaler), CA (Cluster Autoscaler), and CPA (Custom Pod Autoscaler). In this article, we will explore these categories, focusing on three aspects:

1.  Objective
2.  Trigger Conditions
3.  Adjustment Targets

We will delve into these three dimensions for each of these mechanisms.

## Objective

Deployment/ReplicaSet can deploy multiple replicas of Pods, but having a fixed number lacks flexibility, especially when application traffic fluctuates based on specific periods. In scenarios like this, you can use [HPA (Horizontal Pod Autoscaler)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) to dynamically adjust the number of Pods.

## Trigger Conditions

HPA is a built-in controller in Kubernetes. It communicates with the API Server to determine whether to adjust the number of Pods (increase or decrease). When Metrics Server is installed in the environment, it can utilize resource usage metrics like CPU/Memory to make decisions. These metrics are compared with CPU/Memory Requests configured within Pods to determine if the threshold is exceeded. Additionally, usage can be calculated based on overall Pod usage or specific Container usage.

## Adjustment Targets

HPA adjusts the number of Pods. There are several parameters, including Behavior, that can be adjusted, allowing you to specify the percentage or absolute value by which the number of Pods should change with each adjustment.

> In addition to the default resource usage, HPA can also incorporate metrics or projects like KEDA([https://keda.sh/](https://keda.sh/)) to provide decisions from different perspectives.

## Objective

Unlike HPA, which scales the number of replicas horizontally to handle traffic, [VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) adjusts the resource usage of individual instances, such as CPU and memory. When deploying new applications to Kubernetes, it’s common to struggle with configuring Resource Request/Limit settings. VPA continuously observes the resource usage of instances and performs related operations. These operations can involve adjusting settings and restarting Pods or simply providing suggestions without restarting Pods. The latter relies on operators to collect and modify deployment files based on observed resource usage.

## Trigger Conditions

Once the VPA Controller is deployed in the environment, you can create a VPA to specify which Deployments need to be observed. VPA primarily focuses on observing and calculating appropriate numbers for CPU/Memory Request/Limit settings. Observing this requires time to converge, and results got based on a too-short collection time may result in inappropriate usage estimates.

## Adjustment Targets

VPA operates on a per-Pod basis. It does not modify the number of Pod replicas but estimates the CPU/Memory Request/Limit usage. Under **Auto/Recreate** mode, the corresponding values are set, and the Pod is restarted. In **Off** mode, only calculations are performed without restarting the Pod.

## Objective

HPA and VPA are common methods for managing resource usage, adjusting applications based on horizontal and vertical aspects to handle current demands. [CPA](https://github.com/kubernetes-sigs/cluster-proportional-autoscaler) aims to horizontally scale the number of Pod replicas based on the cluster’s scale. A common example is DNS services. CPA can dynamically adjust the number of DNS instances based on the current cluster scale, which can be either the number of nodes or the overall CPU capacity.

## Trigger Conditions

Unlike HPA/VPA, which focus on the resource usage of the application itself, CPA’s trigger adjustments are based on the node’s own capabilities. The setup starts from the perspective of the application, exploring how many node instances or total CPU instances each replica can handle. Relevant settings include `coresPerReplica` and `nodesPerReplica`. The current suitable number of Pods is calculated using the following formula:

> replicas = max(ceil(cores \* 1/coresPerReplica), ceil(nodes \* 1/nodesPerReplica))

## Adjustment Targets

CPA calculates the suitable number based on the configured `coresPerReplica` and `nodesPerReplica`, as well as the current node scale. It dynamically adjusts the target Pod replicas.

## Objective

Previous methods like HPA, VPA, and CPA adjust the number of Pods dynamically based on various conditions. [CA](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler), on the other hand, dynamically adjusts the number of nodes based on specific conditions. For instance, when the resources on all nodes are fully utilized by Pods, leaving no CPU/Memory resources for new deployments, CA dynamically adds new nodes to provide additional computational resources. Conversely, when node resource usage is low, it can remove nodes dynamically, especially in cloud environments to save costs.

During node removal, the common practice is to use a method similar to Drain. It’s essential to pay attention to parameters like *PodDisruptionBudget* and *terminationGracePeriodSeconds* to ensure minimal impact on existing services during the application transition.

> The successful completion of the Drain command depends on whether all Pods on the node are successfully removed. If there are Pods that require an extended time (*terminationGracePeriodSeconds*) to handle the grafecul shutdown process, the timing of node eviction depends on whether those Pods terminate smoothly.

## Trigger Conditions

A common triggering scenario is when any Pod goes into a Pending state due to insufficient resources on the k8s cluster. This action prompts the CA Controller to add a new node. Once the new node is successfully added to the Kubernetes cluster and becomes Ready, the application can be deployed and run smoothly. Conversely, when node usage falls below a threshold for a certain period, Pods on the target node can be moved and the node removed.

> Different Kubernetes platforms have varied implementations, so it’s necessary to confirm the specific implementation and related settings, such as evenly distributing new nodes across different zones or using annotations to prevent specific applications from being evicted. All settings are platform-dependent.

## Adjustment Targets

CA adjusts on a per-node basis. When a node is removed, all running Pods are rescheduled to other nodes.

1.  All the mechanisms mentioned above are not mutually exclusive. For instance, an application category can use HPA to adjust the Pod count and complement it with CA to dynamically adjust the node count to meet the requirements.
2.  Due to the increase or decrease in the number of Pods and nodes caused by these operations, unexpected Pod distribution scenarios may arise. In such cases, mechanisms like [descheduler](https://github.com/kubernetes-sigs/descheduler) or [Affinity, SpreadConstraint](https://medium.com/me/stats/post/e52eebb4bc38) might be necessary to balance the deployment situation.

The differences between the four mechanisms mentioned above are summarized in the diagram below:

![[Raw/Media/Resources/22e6d2263802e278c5d60b1c3cc7d945_MD5.png]]

1.  [Horizontal Pod Autoscaler (HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
2.  [Vertical Pod Autoscaler (VPA)](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
3.  [Cluster Proportional Autoscaler (CPA)](https://github.com/kubernetes-sigs/cluster-proportional-autoscaler)
4.  [Cluster Autoscaler (CA)](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)