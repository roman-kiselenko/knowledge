---
title: "Unleashing the Power of Kubernetes: Deploying Containers with Cluster Resource Access"
source: https://itnext.io/unleashing-the-power-of-kubernetes-deploying-containers-with-cluster-resource-access-ee2cef29e24e
clipped: 2023-11-10
published: 
category: k8s
tags:
  - containers
read: false
---

*A Guide to Optimizing Container Deployment with Cluster Resource Accessibility on Kubernetes*

![[Raw/Media/Resources/661deda554f7ccb815ff9d0087c84b50_MD5.png]]

Managing Kubernetes cluster resources and applications from a pod

**by** [**Franco Stellari**](https://medium.com/@francostellari)

Did you ever wonder how applications like [Argo CD](https://argoproj.github.io/cd/), [Kyverno](https://kyverno.io/), and [KCP](https://kcp.io/) running in a [Kubernetes](https://kubernetes.io/) pod can control and manage other applications running in the cluster? [*Role*, *Role Binding*, and *Service Account*](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) access resources allow one to manage the entire cluster, its resources, and the life cycle of application. For example, you may want to develop a custom solution for managing cluster for [disconnected applications at the edge](https://github.com/kcp-dev/edge-mc).

Recently, I have been working on the [KCP-Edge](https://github.com/kcp-dev/edge-mc) project focused on multi-cluster application lifecycle management at the edge, where scale, connectivity, and security are very important. Experimenting with KCP, one can learn that the [KCP Sync](https://github.com/kcp-dev/kcp) component is currently not capable of synchronizing some types Kubernetes resources, specifically cluster resources. For this reason, I decided to learn more about Kubernetes access resources.

In this blog, I share my experience and discuss how to setup the proper [*Role*, *Role Binding*, and *Service Account*](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) for an applications to access a Kubernetes cluster. Later, I show multiple scenarios demonstrating how to assign permissions to access the resources in a Kubernetes cluster and allow *kubectl* commands to run from a pod. Finally, I show how to create a simple containerized application using [*kubectl*](https://kubernetes.io/docs/reference/kubectl/) to explore and manage the local cluster, such as listing all the pods running in the Kubernetes cluster, as well as start a new *nginx* application, representing a target application being managed by another pod. In this post, the simple *nginx* application is chosen to keep the focus on the resource access management, but in the future we plan to further apply this learning to more interesting applications requiring cluster level resource access, such as [Argo CD](https://argoproj.github.io/cd/), [Kyverno](https://kyverno.io/), and [KCP-Edge](https://github.com/kcp-dev/edge-mc).

This blog is part of a series of posts from the [KCP-Edge](https://github.com/kcp-dev/edge-mc) community regarding challenges related to multi-cloud and edge. You can learn more about the challenges and read posts from other members of the KCP-Edge community on edge and multi-cloud topics [here](https://medium.com/@clubanderson/navigating-the-edge-overcoming-multi-cloud-challenges-in-edge-computing-182e13255e29).

In most cases, applications and services deployed on a Kubernetes cluster are maintained in isolated pods and, for security reasons, are not allowed to interact with the Kubernetes cluster itself.

However, in some cases, one may want to develop local applications with the purpose of managing the cluster.  
This may be useful when:

-   the complexity of the cluster increase and human intervention is not sufficient, or
-   when the cluster is not externally accessible (*e.g.*, behind a firewall) or is intermittently connected (*e.g.*, an edge device) and therefore self managing capability are important.

For this purpose, the application or service may be assigned additional *Roles* and *Role Bindings*, and *Service Account* to enable it to access, deploy, and manage other applications in the Kubernetes cluster or the cluster itself.

## Service Account

From [Kubernetes documentation](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/) we learn that a *Service Account* is a standard Kubernetes object that is used to provides an identity for apps/processes that run in a Pod.

An app/process inside a Pod can use the identity of its associated Service Account to authenticate to the cluster’s API server.

*Service Account* can only be namespaced objects. Therefore, applications/services in a given namespace can be associated and use *Service Accounts* that are created in the same namespace.

The code snippet below shows the creation of a custom namespace called `app-namespace`, where the application will be deployed, and the creation of a `app-service-account` in the same namespace:

```yaml
apiVersion: v1  
kind: Namespace  
metadata:  
  name: app-namespace  
\---  
apiVersion: v1  
kind: ServiceAccount  
metadata:  
  name: app-service-account  
  namespace: app-namespace
```

Later, the application will be associated to the `app-service-account` and it will be able to use the *Roles* that are bound to the service account.

## Role and Cluster Role

From [Kubernetes documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) we learn that *Role* or *Cluster Role* are standard Kubernetes objects that contain rules representing a set of permissions that can be assigned to a *Service Account*.

The main difference between a *Role* and a *Cluster Role* is that the former is associated to a specific *namespace*, thus providing rules that are intrinsically limited to such namespace, while the latter is not associated to a *namespace* and it can therefore contain rules to manage the entire Cluster. For example, a *Cluster Role* can be used by a namespaced app/service to access:

-   cluster level resources (*i.e.*, *nodes*, *namespaces*)
-   non-resource endpoints (*i.e.*, `/healthz`)
-   as well as, namespaced resources (*i.e.*, *pods*, *apps*)

The code snippet below shows an example of *Cluster Role* definition: a new `app-cluster-role` object is created and assigned `create`, `get`, and `list` capabilities for all types of resources in the Kubernetes cluster (*i.e.*, by using `"*"`).

```yaml
apiVersion: rbac.authorization.k8s.io/v1  
kind: ClusterRole  
metadata:  
  name: app-cluster-role  
rules:  
\- apiGroups: \["\*"\]   
  resources: \["\*"\]   
  verbs: \["create", "get", "list"\]
```

Limited access to specific resources can be achieved by naming Kubernetes objects such as *node*, *namespace*, *pod*, *deployment*, *etc*.

Unlimited actions can be granted by specifying `"*"` for the `verbs`.

## Role Binding and Cluster Role Binding

From [Kubernetes documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) we learn that *Role Binding* or *Cluster Role Binding* are standard Kubernetes objects that are used to connect rules defined in *Roles* or *Cluster Roles* objects to *Users* or *Service Accounts*.

The code snippet below shows an example of *Cluster Role Binding* definition: the previously created *Cluster Role* is associated to the *Service Account*. A new `app-cluster-role-binding` object is created connecting the `app-service-account` in the `app-namespace` *namespace* to the rules in the `app-cluster-role` *cluster role*.

```yaml
apiVersion: rbac.authorization.k8s.io/v1  
kind: ClusterRoleBinding  
metadata:  
  name: app-cluster-role-binding  
subjects:  
\- kind: ServiceAccount  
  name: app-service-account  
  namespace: app-namespace  
  apiGroup: ""  
roleRef:  
  kind: ClusterRole  
  name: app-cluster-role  
  apiGroup: ""
```

As previously discussed, a *Service Account* is always a namespaced resource that must reside in the same *Namespace* of the pod/application/deployment referencing it.

On the other hand, namespaced and cluster-level *Roles* and *Role Bindings* can be combined in different configurations following these limitations:

-   when using *Roles* and *Role Bindings*, they must exist in the same *namespace*, possibly different than the *Service Account* *namespace*, and they grant access to the resources of the *Role* *namespace*
-   when a *Role Binding* reference a namespaced *Role*, the service account will only have access to the resources in the *Role* *namespace*
-   *Cluster Role Bindings* are used in combination with *Cluster Roles* to grant access across all resources and *namespaces*
-   *Cluster Role Bindings* can not reference namespaced *Roles*

In these section, we will discuss four application scenarios where different combinations of *Roles* and *Role Bindings* are used to achieve different results.

## Scenario 1

![[Raw/Media/Resources/1135602f365bacd785e46c906f34198c_MD5.png]]

Scenario 1

A *Role* `app-role` and a *Role Binding* `app-role-binding` are created in the same *namespace* `app-namespace` of the *Service Account* `app-service-account` and application. This is the simplest configuration that is used to assign rules limited to the resources in the same *namespace* `app-namespace` of the application. In this case, the *app* can use the resources defined in `app-role` exclusively inside the the same *namespace*. The *app* does not have access to resources in other namespaces or at the cluster level.

```yaml
apiVersion: v1  
kind: Namespace  
metadata:  
  name: app-namespace  
\---  
apiVersion: v1  
kind: ServiceAccount  
metadata:  
  name: app-service-account  
  namespace: app-namespace  
\---  
apiVersion: rbac.authorization.k8s.io/v1  
kind: Role  
metadata:  
  name: app-role  
  namespace: app-namespace  
rules:  
\- apiGroups: \["\*"\]   
  resources: \["\*"\]   
  verbs: \["\*"\]   
\---  
apiVersion: rbac.authorization.k8s.io/v1  
kind: RoleBinding  
metadata:  
  name: app-role-binding  
  namespace: app-namespace  
subjects:  
\- kind: ServiceAccount  
  name: app-service-account  
  namespace: app-namespace  
  apiGroup: ""  
roleRef:  
  kind: Role  
  name: app-role  
  apiGroup: ""
```

## Scenario 2

![[Raw/Media/Resources/ad0598cc86e8493f031c1bb9b64b568b_MD5.png]]

Scenario 2

A *Role* `app-role` and a *Role Binding* `app-role-binding` are both created in a different *namespace* `target-namespace` of the *Service Account* `app-service-account` and application. This configuration is used to allow an application in the `app-namespace` *namespace* access to resources in the `target-namespace` *namespace*. In this case, the `app-service-account` must reside in the same `app-namespace` of the *app*, while the `app-role-binding` must be in the `target-namespace`. The *app* does not have access to resources in other namespaces or at the cluster level.

```
apiVersion: v1  
kind: Namespace  
metadata:  
  name: app-namespace  
\---  
apiVersion: v1  
kind: ServiceAccount  
metadata:  
  name: app-service-account  
  namespace: app-namespace  
\---  
apiVersion: v1  
kind: Namespace  
metadata:  
  name: target-namespace  
\---  
apiVersion: rbac.authorization.k8s.io/v1  
kind: Role  
metadata:  
  name: app-role  
  namespace: target-namespace  
rules:  
\- apiGroups: \["\*"\]   
  resources: \["\*"\]   
  verbs: \["\*"\]   
\---  
apiVersion: rbac.authorization.k8s.io/v1  
kind: RoleBinding  
metadata:  
  name: app-role-binding  
  namespace: target-namespace  
subjects:  
\- kind: ServiceAccount  
  name: app-service-account  
  namespace: app-namespace  
  apiGroup: ""  
roleRef:  
  kind: Role  
  name: app-role  
  apiGroup: ""
```

## Scenario 3

![[Raw/Media/Resources/8ff93b325756e131d38480a44422a3a9_MD5.png]]

Scenario 3

A *Cluster Role* `app-cluster-role` is referenced by a namespaced *Role Binding* `app-role-binding` in a different *namespace* `target-namespace` of the *Service Account* `app-service-account` and application. This configuration is used to grant the cluster level rules in `app-cluster-role` to the application in the `app-namespace` *namespace* for the resources in the `target-namespace` *namespace*. This configuration is similar to the previous one, except that the *Role* has been removed from the `target-namespace` *namespace*, thus making it available to other applications. In this case, *app* does not have access to resources in other namespaces (other than `target-namespace`) or at the cluster level even though a is `app-cluster-role` used. The case where `target-namespace == app-namespace` reverts to the first simple scenario.

```yaml
apiVersion: v1  
kind: Namespace  
metadata:  
  name: app-namespace  
\---  
apiVersion: v1  
kind: ServiceAccount  
metadata:  
  name: app-service-account  
  namespace: app-namespace  
\---  
apiVersion: v1  
kind: Namespace  
metadata:  
  name: target-namespace  
\---  
apiVersion: rbac.authorization.k8s.io/v1  
kind: ClusterRole  
metadata:  
  name: app-cluster-role  
rules:  
\- apiGroups: \["\*"\]   
  resources: \["\*"\]   
  verbs: \["\*"\]   
\---  
apiVersion: rbac.authorization.k8s.io/v1  
kind: RoleBinding  
metadata:  
  name: app-role-binding  
  namespace: target-namespace  
subjects:  
\- kind: ServiceAccount  
  name: app-service-account  
  namespace: app-namespace  
  apiGroup: ""  
roleRef:  
  kind: ClusterRole  
  name: app-cluster-role  
  apiGroup: ""
```

## Scenario 4

![[Raw/Media/Resources/7eed661f6c0cf1feceaf11699e95155b_MD5.png]]

Scenario 4

A *Cluster Role* `app-cluster-role` and *Cluster Role Binding* `app-cluster-role-binding` are associated to the *Service Account* `app-service-account` and application. This configuration is used to allow the application in the `app-namespace` *namespace* access to the cluster resources, including applications in other namespaces.

```yaml
apiVersion: v1  
kind: Namespace  
metadata:  
  name: app-namespace  
\---  
apiVersion: v1  
kind: ServiceAccount  
metadata:  
  name: app-service-account  
  namespace: app-namespace  
\---  
apiVersion: rbac.authorization.k8s.io/v1  
kind: ClusterRole  
metadata:  
  name: app-cluster-role  
rules:  
\- apiGroups: \["\*"\]   
  resources: \["\*"\]   
  verbs: \["\*"\]   
\---  
apiVersion: rbac.authorization.k8s.io/v1  
kind: ClusterRoleBinding  
metadata:  
  name: app-cluster-role-binding  
subjects:  
\- kind: ServiceAccount  
  name: app-service-account  
  namespace: app-namespace  
  apiGroup: ""  
roleRef:  
  kind: ClusterRole  
  name: app-cluster-role  
  apiGroup: ""
```

First, create a simple *Ubuntu* container image with *kubectl* using the `Dockerfile` below:

```Dockerfile
FROM ubuntu  
  
ARG TARGETPLATFORM=linux/amd64  
RUN apt update \\  
    && apt install -y curl \\  
    && curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/${TARGETPLATFORM}/kubectl \\  
    && chmod +x ./kubectl \\  
    && mv ./kubectl /usr/local/bin/kubectl
```

Second, build the image `app-image` using the command:

```shell
if \[ "$(arch)" = "aarch64" \]  
then  
    docker build --build-arg TARGETPLATFORM=linux/arm64 -t app-image .  
else  
    docker build --build-arg TARGETPLATFORM=linux/amd64 -t app-image .  
fi
```

Third, deploy the `app` using the following `app.yaml` yaml:

  
```yaml
apiVersion: v1  
kind: Namespace  
metadata:  
  name: app-namespace  
\---  
  
apiVersion: v1  
kind: ServiceAccount  
metadata:  
  name: app-service-account  
  namespace: app-namespace  
\---  
  
apiVersion: rbac.authorization.k8s.io/v1  
kind: ClusterRole  
metadata:  
  name: app-cluster-role  
rules:  
\- apiGroups: \["\*"\]  
  resources: \["\*"\]   
  verbs: \["\*"\]       
\---  
  
apiVersion: rbac.authorization.k8s.io/v1  
kind: ClusterRoleBinding  
metadata:  
  name: app-cluster-role-binding  
subjects:  
\- kind: ServiceAccount  
  name: app-service-account  
  namespace: app-namespace  
  apiGroup: ""  
roleRef:  
  kind: ClusterRole  
  name: app-cluster-role  
  apiGroup: ""  
\---  
  
apiVersion: v1  
kind: Pod  
metadata:  
  name: app  
  namespace: app-namespace  
spec:  
  serviceAccountName: app-service-account  
  restartPolicy: Never   
  containers:  
    \- name: app  
      image: app-image  
      imagePullPolicy: IfNotPresent   
      command: \["/bin/sh"\]  
      args:  
        \- \-c  
        \- \>-  
            echo Check initial cluster status:  
            && echo  
            && kubectl get namespaces  
            && echo  
            && kubectl get pods -A  
            && echo  
            && echo Deploy nginx in an custom-ns namespace:  
            && echo  
            && kubectl create namespace custom-ns  
            && kubectl create -f https://k8s.io/examples/application/deployment.yaml --namespace custom-ns  
            && sleep 30  
            && echo  
            && echo Check cluster status after deployment:  
            && echo  
            && kubectl get namespaces  
            && echo  
            && kubectl get pods -A  
            && echo  
            && echo Cleaning up:  
            && echo  
            && kubectl delete -f https://k8s.io/examples/application/deployment.yaml --namespace custom-ns  
            && kubectl delete namespace custom-ns  
            && sleep 30  
            && echo  
            && echo Check cluster status after cleanup:  
            && echo  
            && kubectl get namespaces  
            && echo  
            && kubectl get pods -A

```
Deploy the yaml with the command:

```
kubectl apply -f app.yaml

namespace/app-namespace created  
serviceaccount/app-service-account created  
clusterrole.rbac.authorization.k8s.io/app-cluster-role created  
clusterrolebinding.rbac.authorization.k8s.io/app-cluster-role-binding created
```

Finally, check the results `app` in the logs using the command:

```
kubectl logs -n app-namespace app
```

Check initial cluster status:
```

NAME                            STATUS   AGE  
default                         Active   244d  
kube-node-lease                 Active   244d  
kube-public                     Active   244d  
kube-system                     Active   244d  
kubevirt-hostpath-provisioner   Active   244d  
openshift                       Active   244d  
openshift-controller-manager    Active   244d  
openshift-dns                   Active   244d  
openshift-infra                 Active   244d  
openshift-ingress               Active   244d  
openshift-node                  Active   244d  
openshift-service-ca            Active   244d  
app-namespace                   Active   2s

NAMESPACE                       NAME                                  READY   STATUS    RESTARTS   AGE  
kube-system                     kube-flannel-ds-sm89k                 1/1     Running   22         244d  
kubevirt-hostpath-provisioner   kubevirt-hostpath-provisioner-9tj5d   1/1     Running   3          107d  
openshift-dns                   dns-default-7jm9f                     2/2     Running   44         244d  
openshift-dns                   node-resolver-j6vzs                   1/1     Running   22         244d  
openshift-ingress               router-default-85bcfdd948-drqjw       1/1     Running   22         244d  
openshift-service-ca            service-ca-7764c85869-zh4kj           1/1     Running   4520       244d  
app-namespace                   app                                   1/1     Running   0          2s

Deploy nginx in an custom-ns namespace:

namespace/custom-ns created  
deployment.apps/nginx-deployment created

Check cluster status after deployment:

NAME                            STATUS   AGE  
custom-ns                       Active   32s  
default                         Active   244d  
kube-node-lease                 Active   244d  
kube-public                     Active   244d  
kube-system                     Active   244d  
kubevirt-hostpath-provisioner   Active   244d  
openshift                       Active   244d  
openshift-controller-manager    Active   244d  
openshift-dns                   Active   244d  
openshift-infra                 Active   244d  
openshift-ingress               Active   244d  
openshift-node                  Active   244d  
openshift-service-ca            Active   244d  
app-namespace                   Active   34s

NAMESPACE                       NAME                                  READY   STATUS    RESTARTS   AGE  
custom-ns                       nginx-deployment-66b6c48dd5-jcsf5     1/1     Running   0          31s  
custom-ns                       nginx-deployment-66b6c48dd5-tfj5c     1/1     Running   0          31s  
kube-system                     kube-flannel-ds-sm89k                 1/1     Running   22         244d  
kubevirt-hostpath-provisioner   kubevirt-hostpath-provisioner-9tj5d   1/1     Running   3          107d  
openshift-dns                   dns-default-7jm9f                     2/2     Running   44         244d  
openshift-dns                   node-resolver-j6vzs                   1/1     Running   22         244d  
openshift-ingress               router-default-85bcfdd948-drqjw       1/1     Running   22         244d  
openshift-service-ca            service-ca-7764c85869-zh4kj           1/1     Running   4520       244d  
app-namespace                   app                                   1/1     Running   0          34s
```

Cleaning up:

deployment.apps "nginx-deployment" deleted  
namespace "custom-ns" deleted

```
Check cluster status after cleanup:

NAME                            STATUS   AGE  
default                         Active   244d  
kube-node-lease                 Active   244d  
kube-public                     Active   244d  
kube-system                     Active   244d  
kubevirt-hostpath-provisioner   Active   244d  
openshift                       Active   244d  
openshift-controller-manager    Active   244d  
openshift-dns                   Active   244d  
openshift-infra                 Active   244d  
openshift-ingress               Active   244d  
openshift-node                  Active   244d  
openshift-service-ca            Active   244d  
app-namespace                   Active   71s

NAMESPACE                       NAME                                  READY   STATUS    RESTARTS   AGE  
kube-system                     kube-flannel-ds-sm89k                 1/1     Running   22         244d  
kubevirt-hostpath-provisioner   kubevirt-hostpath-provisioner-9tj5d   1/1     Running   3          107d  
openshift-dns                   dns-default-7jm9f                     2/2     Running   44         244d  
openshift-dns                   node-resolver-j6vzs                   1/1     Running   22         244d  
openshift-ingress               router-default-85bcfdd948-drqjw       1/1     Running   22         244d  
openshift-service-ca            service-ca-7764c85869-zh4kj           1/1     Running   4520       244d  
app-namespace                   app                                   1/1     Running   0          71s
```

In this blog we have shown how to use *Cluster Role*, *Cluster Role Binding*, and *Service Account* to deploy a simple application capable of accessing the resources of the cluster using *kubectl* from within a pod.

This blog is part of a series of posts from the [KCP-Edge](https://github.com/kcp-dev/edge-mc) community regarding challenges related to multi-cloud and edge. You can learn more about the challenges and read posts from other members of the KCP-Edge community on edge and multi-cloud topics [here](https://medium.com/@clubanderson/navigating-the-edge-overcoming-multi-cloud-challenges-in-edge-computing-182e13255e29).