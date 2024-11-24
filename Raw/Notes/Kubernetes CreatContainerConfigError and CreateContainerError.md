---
title: Kubernetes CreatContainerConfigError and CreateContainerError
source: https://sysdig.com/blog/kubernetes-createcontainerconfigerror-createcontainererror/
clipped: 2023-09-04
published: 
category: k8s
tags:
  - k8s
read: true
---

CreateContainerConfigError and CreateContainerError are two of the most prevalent Kubernetes errors found in cloud-native applications.

**CreateContainerConfigError** is an error happening when the **configuration specified for a container in a Pod is not correct** or is missing a vital part.

**CreateContainerError** is a problem happening **at a later stage** in the container creation flow. Kubernetes displays this error when it attempts to create the container in the Pod.

## What is CreateContainerConfigError?

During the process to start a new container, Kubernetes first tries to generate the configuration for it. In fact, this is handled internally by calling a method called *generateContainerConfig*, which will try to retrieve:

-   Container command and arguments
-   Relevant persistent volumes for the container
-   Relevant ConfigMaps for the container
-   Relevant secrets for the container

Any problem in the elements above will result in a CreateContainerConfigError.

## What is CreateContainerError?

<mark style="background: #FF5582A6;">Kubernetes throws a CreateContainerError when there’s a problem in the creation of the container, but unrelated with configuration, like a referenced volume not being accessible or a container name already being used.</mark>

*Similar to other problems like [CrashLoopBackOff](https://sysdig.com/blog/debug-kubernetes-crashloopbackoff/), this article only covers the most common causes, but there are many others depending on your current application.*

## How you can detect CreateContainerConfigError and CreateContainerError

As you can see from this output:

-   Pod is not ready: container has an error.
-   There are no restarts: these two errors are not like CrashLoopBackOff, where automatic retrials are in place.

## Kubernetes container creation flow

In order to understand CreateContainerError and CreateContainerConfligError, we need to first know the exact flow for container creation.

Kubernetes follows the next steps every time a new container needs to be started:

1.  Pull the image.
2.  Generate container configuration.
3.  Precreate container.
4.  Create container.
5.  Pre-start container.
6.  Start container.

As you can see, steps 2 and 4 are where a CreateContainerConfig and CreateContainerErorr might appear, respectively.

[![[Raw/Media/Resources/9086d57791026cbc816346a53c0dc784_MD5.png]]](https://sysdig.com/wp-content/uploads/Createcontainererror-02-1170x464.png)

## Common causes for CreateContainerError and CreateContainerConfigError

### Not found ConfigMap

Kubernetes [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) are a key element to store non-confidential information to be used by Pods as key-value pairs.

When adding a ConfigMap reference in a Pod, you are effectively indicating that it should retrieve specific data from it. But, if a Pod references a non-existent ConfigMap, Kubernetes will return a CreateContainerConfigError.

### Not found Secret

Secrets are a more secure manner to store sensitive information in Kubernetes. Remember, though, this is just raw data encoded in base64, so it’s not really encrypted, just obfuscated.

In case a Pod contains a reference to a non-existent secret, Kubelet will throw a CreateContainerConfigError, indicating that necessary data couldn’t be retrieved in order to form container config.

### Container name already in use

While an unusual situation, in some cases a conflict might occur because a particular container name is already being used. Since every docker container should have a unique name, you will need to either delete the original or rename the new one being created.

## How to detect CreateContainerConfigError and CreateContainerError in Prometheus

When using Prometheus + kube-state-metrics, you can quickly retrieve Pods that have containers with errors at creation or config steps:

```
kube_pod_container_status_waiting_reason{reason="CreateContainerConfigError"} > 0
```

```
kube_pod_container_status_waiting_reason{reason="CreateContainerError"} > 0
```

![[Raw/Media/Resources/9abc8d1c7c7e84e7488367a5327f85f9_MD5.png]]

## Other similar errors

### Pending

[Pending is a Pod status](https://sysdig.com/blog/kubernetes-pod-pending-problems/) that appears when the Pod couldn’t even be started. Note that this happens at schedule time, so Kube-scheduler couldn’t find a node because of not enough resources or not proper taints/tolerations config.

### ContainerCreating

ContainerCreating is another waiting status reason that can happen when the container could not be started because of a problem in the execution (e.g: `No command specified`)

`Error from server (BadRequest): container "mycontainer" in pod "mypod" is waiting to start: ContainerCreating` 

### RunContainerError

This might be a similar situation to CreateContainerError, but note that this happens during the run step and not the container creation step.

<mark style="background: #FF5582A6;">A RunContainerError most likely points to problems happening at runtime, like attempts to write on a read-only volume.</mark>

### CrashLoopBackOff

Remember that [CrashLoopBackOff](https://sysdig.com/blog/debug-kubernetes-crashloopbackoff/) is not technically an error, but the waiting time grace period that is added between retrials.

Unlike CrashLoopBackOff events, CreateContainerError and CreateContainerConfigError won’t be retried automatically.

## Conclusion

In this article, you have seen how both CreateContainerConfigError and CreateContainerError are important messages in the Kubernetes container creation process. Being able to detect them and understand at which stage they are happening is crucial for the day-to-day debugging of cloud-native services.

Also, it’s important to know the internal behavior of the Kubernetes container creation flow and what is errors might appear at each step.

Finally, CreateContainerConfigError and CreateContainerError might be mistaken with other different Kubernetes errors, but these two happen at container creation stage and they are not automatically retried.
