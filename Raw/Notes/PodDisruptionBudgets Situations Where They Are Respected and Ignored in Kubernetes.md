---
title: "PodDisruptionBudgets: Situations Where They Are Respected and Ignored in Kubernetes"
source: https://medium.com/@xpiotrkleban/poddisruptionbudgets-situations-where-they-are-respected-and-ignored-in-kubernetes-29ebaabac175
clipped: 2024-12-23
published: 
category: k8s
tags:
  - containers
  - sig-scheduling
read: false
---
![[Raw/Media/Resources/7b6d9765ac690562c7ac52c59f1f9730_MD5.jpg]]

The cluster’s functionality usually relies on some critical services, such as ingress and message broker, that enable the communication and data flow of the applications running on the cluster. Without them, nothing would work as expected. They impact on the **whole functionality of the cluster.** For example**,** without ingress and message broker, your applications may not be able to receive or send data to the outside world or to each other, which can affect the functionality and availability of the cluster. If all replicas of message broker are down, then internal communication may be interrupted, lost or delayed until a **new replica of message broker** is created and becomes ready. This can cause data inconsistency, corruption, or loss for your applications until the message broker is restored.

We need to make sure to have high availability, so ability to recover from failures and sustain performance.

A PDB (Pod Disruption Budget) is an object that defines the availability requirements of an application during a **voluntary disruption**, such as node maintenance, upgrade, or scaling. PDB is another tool that helps to achieve high availability.

We will check if `HPA` or `Deployment` interact with **Pod Disruption Budget** when pods are scaled down or updated. We will also `taint` or `drain` nodes and use Blue-Green for EKS worker nodes.

Table of Contents

· [Concepts](#e35e)  
∘ [PDB object](#f899)  
∘ [Voluntary disruptions and non-voluntary disruptions](#5447)  
· [Commands that cause voluntary disruptions](#bf4d)  
∘ [kubectl cordon](#9206)  
∘ [kubectl drain](#c95d)  
∘ [kubectl taint](#a5b6)  
· [Respect of Disruption Budget by different kubectl commands](#ef6e)  
∘ [Disruption Budget maxUnavailable and Deployment maxUnavailable](#fb11)  
∘ [Disruption Budget and kubectl taint and kubectl drain](#379e)  
∘ [Disruption Budget and HPA minReplicas](#10c6)  
∘ [Updating Kubernetes nodes](#8bb6)  
∘ [EKS managed node group instance change](#15eb)  
∘ [EKS Nodes Group blue-green deployment](#24aa)  
∘ [EKS managed Node Group upgrade initiated by eksctl](#1831)  
· [Conclusions](#cbf6)

Suppose we have three nodes: `node-1`, `node-2,` and `node-3`. We have a **single replica** of `Ingress controller` running on `node-1`. We want to update the instance type (e.g., faster CPU) of `node-1`, so we use `kubectl drain` to drain it. Here is what may happen:

-   `kubectl drain` evicts the pod of `Ingress Controller` from `node-1`. If we don’t have a PDB for our Ingress Controller service, then `kubectl drain` deletes the pod without checking if there is another pod available on `node-2` or `node-3`.
-   Without an ingress controller pod running on our cluster, **our external traffic does not reach our services**, causing errors. This causes downtime or errors for your users or clients until a new pod of ingress is created and becomes ready.
-   To avoid this situation, we should have another Pod of Ingress Controller running on `node-2` or `node-3` before we drain `node-1`. We should also use a PDB to specify the minimum number of available pods for our `Ingress Controller` service, such as `minAvailable: 1`. This way, `kubectl drain` will not evict the pod of ingress controller from `node-1` unless there is another pod running on `node-2` or `node-3,` showing message:`Cannot evict pod as it would violate the pod’s disruption budget`. We should have more than one Pod of a service running to achieve HA.

## PDB object

The PDB definition **only has two attributes** to control the availability requirements: `minAvailable` or `maxUnavailable`(mutually exclusive). Field`maxUnavailable`tells how many pods can be down and `minAvailable` tells how many pods must be running in a cluster. We use `selector` to choose the pods that are affected by the PDB. If we specify an empty selector `({})`, then **all pods in the namespace are covered by the pod disruption budget.**

It ensures availability of an application during a voluntary disruption, such as node maintenance, upgrade, or scaling. A PDB specifies the number or percentage of pods that must be available. A PDB allows you to limit the disruption to your application and maintain high availability while permitting the cluster administrator to manage the cluster nodes.

Here is an example of a PDB definition in YAML format:

apiVersion: policy/v1  
kind: PodDisruptionBudget  
metadata:  
  name: my-pdb  
spec:  
  selector:  
    matchLabels:  
      app: my-app  
  minAvailable: 2

This PDB ensures that at least two pods with the label `app: my-app` are available at any given time during a voluntary disruption.

For more information, see [here](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.27/#poddisruptionbudget-v1-policy) and [here](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#pod-disruption-budgets).

## Voluntary disruptions and non-voluntary disruptions

Voluntary disruptions and non-voluntary disruptions are two types of events that can affect the availability of pods in a Kubernetes cluster. They have different causes and implications for the application owners and cluster administrators.

Non-voluntary disruptions are unavoidable cases that happen due to hardware or system failures.

Examples are:

-   Hardware failure of the physical machine
-   Cloud provider or hypervisor failure
-   Kernel panic
-   Network partition
-   Power outage

Non-voluntary disruptions are not specific to Kubernetes, and they can happen in any system. They can cause pods to disappear or become unreachable, which can affect the availability and reliability of the application. To mitigate non-voluntary disruptions, application owners can use strategies such as:

-   Redundancy
-   Replicate your application for higher availability
-   Spread your application across AZs

**Voluntary disruptions** are intentional cases that happen due to actions initiated by the application owner or the cluster administrator.

Voluntary disruptions are specific to Kubernetes, and they can be controlled or prevented by using Pod Disruption Budgets (PDBs). **Kubernetes will respect these budgets when performing voluntary disruptions.**

Examples of voluntary disruptions are:

-   Draining a node for repair or upgrade
-   Draining a node from a cluster to scale the cluster down

> Note: **The PDB will not be respected by every action that deletes or restarts a pod!**

For more information about Eviction API, see [here](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.27/#create-eviction-pod-v1-core).  
For more information about disruptions, see [here](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/),

Some Kubernetes administration commands that cause voluntary disruptions are:

`kubectl cordon <node>` marks a node as unschedulable, prevents any new pods from being scheduled on the node

`kubectl drain <node>` evicts all pods on a node except for `DaemonSets` and mirror pods. The deleted pods are rescheduled on other nodes if possible.

`kubectl delete <node>` command deletes a node object from the Kubernetes API server.

`kubectl drain` and `kubectl cordon` help you prepare a node for maintenance, update, or removal.

## kubectl cordon

`kubectl cordon` marks a node as unschedulable, which means it will not accept any new pods. However, it does not affect the existing pods on the node, which will continue to run normally. You can use `kubectl cordon` to prevent new pods from being scheduled on a node that you plan to drain or delete later. For example:

kubectl cordon my\-node

## kubectl drain

`kubectl drain` evicts all the pods from a node, except for the ones that are part of a `DaemonSet`. It also cordons the node automatically, so no new pods can be scheduled on it. The evicted pods will be rescheduled on other nodes in the cluster, as long as there is enough capacity and resources. You can use `kubectl drain` to safely remove all the workloads from a node before you perform maintenance or delete it.

For example:

kubectl drain my\-node

However, typically running a command without extra parameters will fail from draining the node.

`DaemonSet` pods are managed by a **daemonset controller**, which ensures that a copy of a pod runs on every node in the cluster. Draining without `--ignore-daemonsets` will **stop and not continue the process.** It cannot remove any pods that are controlled by a daemon set, because the **daemonset controller** will make new pods right away to replace them.

`DaemonSet` **does not care if the node is unschedulable.**

You can use some flags to modify the behavior of `kubectl drain`, such as:

-   `--ignore-daemonsets`: This flag allows you to ignore **DaemonSet** pods and evict them as well. By default, kubectl drain will not evict DaemonSet pods, because they are expected to run on every node in the cluster. However, if you are deleting the node permanently, you might want to evict them as well.
-   `--delete-emptydir-data`: Remove the contents of emptyDir volumes from the pods that are drained.
-   `--force`: This flag allows you to force the eviction of pods that are not managed by a controller (such as a Deployment or a StatefulSet). By default, kubectl drain will not evict these pods, because they will not be **rescheduled by the controller**. **However, if you are sure that you don’t need these pods, or you have other ways to recreate them, you can use this flag to evict them as well.**

Let’s do a brief recap on the subject of **static pods**. Static pods are pods that are created and managed by the `kubelet` on a specific node, without the API server. Static Pods are always bound to one `kubelet` on a specific node.

For more information about static pods, see [here](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/),

Here is a quick summary of the steps we have to take to run a static pod:

-   Log in to one of the nodes where we want to run the static pod. .
-   Create a pod definition file in YAML format and save it in a directory that is specified by the `kubelet`. The default directory is `/etc/kubernetes/manifests`, For example, you can create a file named `static.yaml` with the following content:

apiVersion: v1  
kind: Pod  
metadata:  
  name: static-pod  
  labels:  
    app: my-app  
spec:  
  containers:  
  \- name: nginx  
    image: nginx

-   Restart the `kubelet` to make it read the pod definition file and create the static pod. You can use the command `sudo systemctl restart kubelet`

## kubectl taint

Taints and tolerations are a way to control which pods can be scheduled on which nodes in Kubernetes. A taint is a key-value pair with an effect that is applied to a node, and a toleration is a key-value pair that matches a taint and allows a pod to be scheduled on a tainted node.

For example, suppose you have a cluster with three nodes: `node1`, `node2`, and `node3`. You want to dedicate `node1` and `node2` for running **GPU-intensive workloads**, and `node3` for running other workloads. You can achieve this by adding a taint to `node1` and `node2`, such as:

kubectl taint nodes node1 gpu=true:NoSchedule  
kubectl taint nodes node2 gpu=true:NoSchedule

This will **prevent any pod that does not have a matching toleration** from being scheduled on `node1` and `node2`. The effect `NoSchedule` means that the scheduler **will not place any pod on the tainted node unless it has a toleration.**

Next, you need to add a toleration to the pods that need to run on the GPU nodes, such as:

apiVersion: v1  
kind: Pod  
metadata:  
  name: gpu-pod  
spec:  
  containers:  
  \- name: gpu-container  
    image: gpu-image  
  tolerations:  
  \- key: "gpu"  
    operator: "Equal"  
    value: "true"  
    effect: "NoSchedule"

This allows the pod to be scheduled on either `node1` or `node2`, but not on `node3`. The toleration matches the taint by having the same key, value, and effect.

You can also use other effects for taints, such as `PreferNoSchedule`, which means that the scheduler will try to avoid placing pods that do not have toleration on the tainted nodes

If we add a taint with `NoSchedule` effect to a node, **then the existing pods on that node will not be evicted,** even if they do not tolerate the taint. If you want to evict the existing pods that do not tolerate the taint, you need to use the `NoExecute` effect instead.

We will look at different situations that can result in pods being evicted or deleted and see what happens to the application replicas.

For that, let’s create a multi-node sandbox cluster. One of the ways to easily start a cluster with multiple nodes (within seconds), is by using `k3d`. `k3d` is a [lightweight wrapper](https://docs.rancherdesktop.io/how-to-guides/create-multi-node-cluster/) to run k3s, a lightweight Kubernetes distribution, in Docker. To start a cluster with 3 nodes, you can use the following command:

`k3d cluster create mycluster --agents 3`

Use the `kubectl config use-context mycluster`command to switch between clusters.

Use `kubectl get nodes` to interact with your new cluster and to confirm that k3s works as expected.

To list all `k3d` clusters, we can use the following command:

`k3d cluster list`

To delete a `k3d` cluster, you can use the following command:

`k3d cluster delete mycluster`

## Disruption Budget maxUnavailable and Deployment maxUnavailable

Both `PDB` and `Deployment` define `maxUnavailable`, however for different purposes. `maxUnavailable` of a `Deployment` object defines how many pods can be terminated at the same time.

apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: nginx-deployment  
spec:  
  selector:  
    matchLabels:  
      app: nginx  
  replicas: 3  
  strategy:  
    type: RollingUpdate  
    rollingUpdate:  
      maxSurge: 0  
      maxUnavailable: 3  
  template:  
    metadata:  
      labels:  
        app: nginx  
    spec:  
      containers:  
      \- name: nginx  
        image: nginx:1.21.0   
        ports:  
        \- containerPort: 80  
          readinessProbe:  
            httpGet:  
              path: /  
              port: 80  
            initialDelaySeconds: 10  
            periodSeconds: 5  
            successThreshold: 1  
            failureThreshold: 3

Define the PDB using the opposite logic to `maxUnavailable` by setting `minAvailable` to 3.

apiVersion: policy/v1  
kind: PodDisruptionBudget  
metadata:  
  name: nginx-pdb  
spec:  
  minAvailable: 3  
  selector:  
    matchLabels:  
      app: nginx 

Once the pods from the above manifests are running, update the image from `nginx:1.21.0` to `nginx:1.21.1`, then apply the changes with `kubectl apply -f <deployment.yaml>` and observe the result with `kubectl describe deploy nginx-deployment`

![[Raw/Media/Resources/2ed9cd9b1990de102484ac60d28c8e43_MD5.png]]

No replicas are running, so the users might experience downtime.

Conclusion: `Deployment` does not care about PDB and will use its own `maxUnavailable` value for rolling updates, so you should be careful when choosing this value to keep HA.

## Disruption Budget and kubectl taint and kubectl drain

If you have a pod disruption budget (PDB) and add a taint with `NoExecute` effect to a node, then the pods that do not tolerate the taint will be evicted **from the node regardless of the PDB.** This means that the PDB will not protect the pods from being disrupted by the taint. The `NoExecute` effect does not use the **eviction API,** which respects the PDB.

Therefore, you should be careful when using taints with `NoExecute` effect, as they may cause more disruptions than expected.

![[Raw/Media/Resources/228c0583ce6dde09d1c2cb43fe30f976_MD5.png]]

![[Raw/Media/Resources/e1f1ce6e32f89bcc7421e03489ee41b8_MD5.png]]

Now let’s find out what happens if we try to drain the node:

kubectl drain k3d-three-node-cluster3-server-1 \--delete-emptydir-data \--ignore-daemonsets

![[Raw/Media/Resources/f0b0b1a1fff18ca4b9426bb1ab51e21c_MD5.png]]

Node drain is unable to complete the **eviction process and tries endlessly.**

Modify `minAvailable: 1` field in PDB:

apiVersion: policy/v1  
kind: PodDisruptionBudget  
metadata:  
  name: nginx-pdb  
spec:  
  minAvailable: 1  
  selector:  
    matchLabels:  
      app: nginx 

This way, `kubectl drain` will complete the draining process of a node, as expected.

![[Raw/Media/Resources/ac1b3bfe4d5d8497cfb2f5adaac4f2f4_MD5.png]]

## Disruption Budget and HPA minReplicas

First use `kubectl top node` to make sure that **Kubernetes Metrics Server** is up and running, remove `replicas: 3` from `Deployment`and run HPA in the cluster:

apiVersion: policy/v1  
kind: PodDisruptionBudget   
metadata:  
  name: nginx-pdb  
spec:  
  minAvailable: 3   
  selector:  
    matchLabels:  
      app: nginx   
\---  
apiVersion: autoscaling/v2  
kind: HorizontalPodAutoscaler  
metadata:  
  name: nginx-hpa  
spec:  
  scaleTargetRef:  
    apiVersion: apps/v1  
    kind: Deployment  
    name: nginx-deployment  
  minReplicas: 3  
  maxReplicas: 10  
  metrics:  
  \- type: Resource  
    resource:  
      name: cpu  
      target:  
        type: Utilization  
        averageUtilization: 50

We can additionally verify PDB by using `kubectl describe pdb ngnix-pdb`:

![[Raw/Media/Resources/41574bc6062ac1c48a0fa6d91a5697cc_MD5.png]]

Now change `minReplicas` to 2 and see what happens:

![[Raw/Media/Resources/4a7ce503354397f03595866257c488b2_MD5.png]]

Of course, even if HPA does not respect PDB, it is still beneficial to use PDB in case of node drain, as it can prevent disruptions to the pods and ensure availability.

## Updating Kubernetes nodes

Now, let’s see the outcome of updating the EKS worker nodes.

## EKS managed node group instance change

If you use `aws_eks_node_group` to manage your EKS nodes, you can update the node group configuration by changing the Terraform resource attributes and applying the changes. For example, if we want to change the **instance type** of the node group, we can modify the `instance_types` or `scaling_config` attributes of the `aws_eks_node_group` resource. For example:

resource "aws\_eks\_node\_group" "example" {  
  cluster\_name    = aws\_eks\_cluster.example.name  
  node\_group\_name = "example"  
  node\_role\_arn   = aws\_iam\_role.example.arn  
  subnet\_ids      = aws\_subnet.example\[\*\].id  
  instance\_types  = \["t3.small"\]   
  scaling\_config {  
    desired\_size = 3  
    max\_size     = 6  
    min\_size     = 3  
  }  
}

Change the `instance_types` argument in the `aws_eks_node_group` resource:

resource "aws\_eks\_node\_group" "node-group" {  
  cluster\_name    = aws\_eks\_cluster.cluster.name  
  version         = var.cluster\_version  
  (...)  
  instance\_types = \["t3.medium"\] \# <- change from t3.small to t3.medium

It will trigger a **recreation of the node group**. This means that the existing nodes will be terminated, and new nodes will be launched with the new instance types.

![[Raw/Media/Resources/ce76be9e052d4c78834d4aee17158409_MD5.png]]

The outcome will be similar to simply removing the node group:

![[Raw/Media/Resources/e81de3cf113a956c41940f289be2726c_MD5.png]]

Within a moment, the **ASG** corresponding **Node Group** scales EC2s down to 0.

![[Raw/Media/Resources/ec2976665b3f6365e50eefdf5a19b9a6_MD5.png]]

Terraform will create a new node group, but **pods will experience downtime** (if there are no other Nodes Group with enough capacity)**.**

> Amazon EKS sends a signal to drain the Pods from that node and then waits a few minutes. If the Pods haven’t drained after a few minutes, Amazon EKS lets Auto Scaling continue the termination of the instance.

For more information, see [here](https://docs.aws.amazon.com/eks/latest/userguide/delete-managed-node-group.html).

To prevent that, we can use a blue-green Node Group strategy.

## EKS Nodes Group blue-green deployment

An example of applying blue-green for Node Group EKS in steps:

-   Create a second **Node Group** named green
-   Drain the blue **Node Group** (it respects PDB!)
-   Scale down the blue **Node Group**

Define `green` resource of `aws_eks_node_group`, for example:

resource "aws\_eks\_node\_group" "blue" {  
  cluster\_name    = aws\_eks\_cluster.example.name  
  node\_group\_name = "blue"  
  node\_role\_arn   = aws\_iam\_role.example.arn  
  subnet\_ids      = aws\_subnet.example\[\*\].id  
  scaling\_config {  
    desired\_size = 2  
    max\_size     = 3  
    min\_size     = 1  
  }  
  instance\_types = \["t3.medium"\]   
}

resource "aws\_eks\_node\_group" "green" {  
  cluster\_name    = aws\_eks\_cluster.example.name  
  node\_group\_name = "green"  
  node\_role\_arn   = aws\_iam\_role.example.arn  
  subnet\_ids      = aws\_subnet.example\[\*\].id  
  scaling\_config {  
    desired\_size = 2  
    max\_size     = 3  
    min\_size     = 1  
  }  
  instance\_types = \["t3.large"\]   
}

Apply the Terraform configuration to create both node groups in the cluster.

Use `kubectl drain` command to drain the nodes in the blue node group.

For example:

Suppose we have two **blue** nodes and two **green** nodes in our cluster, and we are running a critical service with three replicas. We want to ensure that at least two replicas are always available, so we create a PodDisruptionBudget (PDB) with `minAvailable: 2`. Let’s say all **3 replicas** are running on `node-0.`

We begin to drain `node-0` and `node-1` via `kubectl drain <node>`(Actually we could start by cordoning instead of draining — if a network problem occurs during the process, some pods might be moved to `node-1`, which we intended to drain. This will result in unnecessary rescheduling)

Both nodes are drained at the same time, `node-1` is drained immediately:

![[Raw/Media/Resources/501e316dd7fa27d7916dc5df57203845_MD5.png]]

As `node-1` is cordoned, it cannot schedule `nginx-deployment` pods (so they are going to be allocated to `node-2` and `node-3` as expected. There is `minAvailable` set to 2 in PDB, so one pod at a time will be evicted.

![[Raw/Media/Resources/2dbc96b193b2c2d1111ade6c9e46e150_MD5.png]]

PODS allocated to `node-0`

![[Raw/Media/Resources/1a5e0e23e1b7c02a069f26ae7486fb29_MD5.png]]

first pod rescheduled to node-2

![[Raw/Media/Resources/5ee0ce3d80e69cd0bcc76d07e810a864_MD5.png]]

second pod reschedules to node-3

![[Raw/Media/Resources/a48952fdd1beb142fa45876d3760b854_MD5.png]]

all three pods rescheduled to Green Node Group

Logs from `node-0` draining:

node/k3d-c-server-0 cordoned  
evicting pod default/nginx-deployment-6dd5d58949-mlndm  
evicting pod default/nginx-deployment-6dd5d58949-ps2kf  
evicting pod default/nginx-deployment-6dd5d58949-pwxgj  
pod/nginx-deployment-6dd5d58949-ps2kf evicted  
error when evicting pods/"nginx-deployment-6dd5d58949-pwxgj" -n "default" (will retry after 5s): Cannot evict pod as it would violate the pod  
evicting pod default/nginx-deployment-6dd5d58949-pwxgj  
(...)  
pod/nginx-deployment-6dd5d58949-pwxgj evicted  
error when evicting pods/"nginx-deployment-6dd5d58949-mlndm" -n "default" (will retry after 5s): Cannot evict pod as it would violate the pod  
evicting pod default/nginx-deployment-6dd5d58949-mlndm  
(...)  
pod/nginx-deployment-6dd5d58949-mlndm evicted  
node/k3d-c-server-0 drained

Now we can scale down the unused node group to zero, for example:

resource "aws\_eks\_node\_group" "blue" {

    
  scaling\_config {  
    desired\_size = 0   
    max\_size     = 0   
    min\_size     = 0   
  }

}

## EKS managed Node Group upgrade initiated by eksctl

Resource `aws_eks_node_group` resource has `update_config` argument, which needs some explanation.

resource "aws\_eks\_node\_group" "example" {  
    
  update\_config {  
    max\_unavailable            = 1  
  }  
}

The `update_config` block only applies to updates initiated by EKS, **such as Kubernetes version updates or AMI changes.** It does not apply to updates initiated by Terraform, such as instance changes.

Initiation of the update can be started by `eksctl` tool with `eksctl upgrade nodegroup` command. It allows updating the Kubernetes version of a node group to match the cluster version. It enables updating EKS node group with zero downtime as a rolling update.

For example:

eksctl upgrade nodegroup \--cluster <cluster> \--name <nodes\_group> \--kubernetes-version <version>

The tool accepts some of the following arguments:

-   `--kubernetes-version` to update the Kubernetes version of a node group to match the cluster version
-   `--release-version` to update of the EKS optimized AMI to use
-   `--force-upgrade` force update regardless of **pod disruption budget**

Argument `update_config` allows us to specify the maximum number of nodes that can be unavailable during an update of the node group. This helps to control availability of the pods running on the nodes.

During the update process, EKS will (by default) respect the Pod Disruption Budget of your workloads.

The `update_config` block parameters:

-   `max_unavailable` – The maximum number of nodes that can be unavailable during the update process. **The default is 1.**
-   `max_unavailable_percentage` – The maximum percentage of nodes that can be unavailable during the update process. The default is 10%.
-   `delay_seconds` – The number of seconds to wait between node updates. **The default is 90 seconds.**

As mentioned `*eksctl upgrade nodegroup*` command can only update the Kubernetes version of a node group and AMI. It cannot update other parameters such as instance type.

To understand the whole update process, see [here](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-update-behavior.html).

We should always test changes in staging environments before applying them to production. This rule also applies to PodDisruptionBudgets (PDBs) because they might not work as expected in some situations. For example, Deployments or Horizontal Pod Autoscalers do not take into account the PDBs of the pods that they replace or scale down. Another example is when we change the instance type of Elastic Kubernetes Service (EKS) node group, which will cause all the EC2 nodes in the group to be terminated and replaced, potentially disrupting our services. In this case, a blue-green deployment strategy might be a better option to avoid downtime. On the other hand, `kubectl drain` does respect PDBs and will only evict pods that are allowed by the PDBs

Thanks