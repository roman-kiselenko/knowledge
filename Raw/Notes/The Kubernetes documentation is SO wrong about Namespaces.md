---
title: The Kubernetes documentation is SO wrong about Namespaces
source: https://pauldally.medium.com/the-kubernetes-documentation-is-wrong-about-namespaces-426beb9f684b
clipped: 2023-10-29
published: 
category: k8s
tags:
  - k8s
read: false
---

Sometimes in life we get incorrect advice. The source of the advice may be well-meaning, but no one is perfect. Point in case — the [documentation for Kubernetes Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) states “For clusters with a few to tens of users, you should not need to create or think about namespaces at all. Start using namespaces when you need the features they provide”.

I would suggest that this is half wrong — yes, you should use Namespaces when you need the features they provide... **However, if you want to run more than one application (or more than one instance of an application) in your cluster you should think about Namespaces, no matter the number of users!**

Imagine that we have only two applications, “A” and “B”.

Application “A”, developed by Jane, consists of:

1.  a microservice API [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
2.  a NoSQL database [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
3.  [Services](https://kubernetes.io/docs/concepts/services-networking/service/) for the API and database Pods
4.  an [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) to route external connections to the API Pods
5.  [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/)s and [Secret](https://kubernetes.io/docs/concepts/configuration/secret/)s, some of which are used by both the Deployment and by the StatefulSet

Application “B”, developed by John, consists of:

1.  a microservice API [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
2.  A [Service](https://kubernetes.io/docs/concepts/services-networking/service/) for the API Pods
3.  [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/)s and [Secret](https://kubernetes.io/docs/concepts/configuration/secret/)s, some of which are used by both the API and by the Deployment

## Namespaces help us avoid naming conflicts

If everything is deployed to the default Namespace, what would happen if both applications use the same name for any of their components? For example, if Jane names application A’s ConfigMap “application.props”, and John also names application B’s ConfigMap “application.props”, they will overwrite each other and the application will not function properly.

You can avoid this by adding a suffix or prefix to every object, but are you sure everyone will do this or that you’ll always be consistant? Namespaces easily prevent this with very little hassle, because objects in different Namespaces can have the same name without conflicting with each other.

And what if we need multiple instances of an application? For example, Application “A” may need different versions running in parallel, and Application “B” may need blue/green, etc. Perhaps you would like to use the same cluster for multiple environments like development and test? Even a single application managed by a single user could easily run into naming conflicts in these scenarios. To use a single Namespace in the same cluster, **all** of these attributes would need to be incorporated into the names of the objects to guarantee uniqueness.

Unfortunately, names of objects in Kubernetes have maximum lengths. Most objects top out at 256 characters, but Pods are restricted to just 64 characters. Putting all of the “instance” or environment information in your object names can impact readability, especially since Kubernetes may start [truncating](https://pauldally.medium.com/why-you-try-to-keep-your-deployment-names-to-47-characters-or-less-1f93a848d34c) your Pod names.

## Namespaces allow us to define granular role-based access control

[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) is defined at the Namespace level, and is relatively simplistic. You can grant access to certain types of resources to different users or groups, but you cannot allow a particular group or user access to only *some* of the resources of a particular type. In a single Namespace, we cannot allow Jane to access only Application A’s Pods, and John to access only Application B’s Pods. In contrast, however, if each application is deployed to their own Namespace, this is easily possible.

Sometimes, you may even want different Namespaces for certain components of the same application. If the NoSQL database in Application “A” isn’t managed by Jane because a central group manages all such databases at your organization, you would probably benefit from multiple Namespaces.

Finally, without different Namespaces and granular access control, what’s to stop Jane from deleting all of Application “B”s Deployments or ConfigMaps?

## Namespaces allow different ResourceQuotas for each application

You might reasonably want to allow application “A” to consume only a certain amount of resources, while allowing Application “B” to consume a different amount of resources. If you run multiple applications in the same Namespace, you may find that one application consumes enough resources that the other application cannot run or performance of that application is impaired.

You could potentially address this problem by carefully configuring the [settings](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) of your individual Pods. However, applying [ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/)s to unique Namespaces for each application is much easier.

## Namespaces make NetworkPolicy more secure and reliable

[NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can be used to increase the security of your application. If Jane determines that only the microservice API Pods should be allowed to communicate with the NoSQL database, she can implement a NetworkPolicy to enforce that. To do this, she would deploy a NetworkPolicy that allows ingress communication only from Pods that have particular labels within the current (default) Namespace.

However, if Application “B” is also in that Namespace, if John decides to use the same labels on his Pods (which Jane really can’t prevent), then the NetworkPolicy that Jane implemented will be more or less useless. In a discrete Namespace, Jane’s NetworkPolicy could allow only Pods **in that same Namespace** that have the particular labels, so that it no longer matters what Pod labels John uses.

These examples only scratch the surface — there are many other use cases and scenarios that could be discussed. Overall, Namespaces can increase the security of your applications, help you avoid outages and generally make your life easier — so even before your clusters have “tens of users”, you should consider using them — precisely because you need the features they provide.