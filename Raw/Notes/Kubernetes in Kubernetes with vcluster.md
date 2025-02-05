---
title: Kubernetes in Kubernetes with vcluster
source: https://blog.devops.dev/kubernetes-in-kubernetes-with-vcluster-a5be97ac5861
clipped: 2023-09-04
published: 
category: k8s
tags:
  - k8s
read: true
---

## Kubernetes inside Kubernetes (N:1-Host-Cluster)

Final State

## Introduction

> Note: with Kubernetes inside Kubernetes I mean a virtual fully functional Kubernetes cluster that runs on top of another cluster (Host).

Imagine you can deploy multiple virtual cluster inside a kubernetes cluster and the most teams would not see any difference at first. You know, this already possible and the solution is called vcluster.

I will not explain how vcluster works in detail and how the individual distros are composed. For this, [lôft](https://loft.sh/) has created a detailed [documentation](https://www.vcluster.com/docs/what-are-virtual-clusters) with examples.

I'm gonna focus on the k3s distro, because it is a solution that works with rootless mode. This is a key factor for me to use it in a production environment. But you can also use k0s, if you prefer it over k3s.

## Main

For the implementation, I use an Azure Kubernetes Service (AKS) with the GitOps approach, which is fulfilled by Argo CD. The used Kubernetes version is 1.24.9. I also use a custom written helm chart as overlay. I will overwrite the helm chart as a subchart from lôft because I want to have a valid cert for my ingress.

## Setup vcluster k3s (rootless mode)

Then let’s start with the hands-on section.

1.  **Install vcluster cli**

  
curl -L -o vcluster "https://github.com/loft-sh/vcluster/releases/latest/download/vcluster-darwin-amd64" && sudo install -c -m 0755 vcluster /usr/local/bin && rm -f vcluster  
  
curl -L -o vcluster "https://github.com/loft-sh/vcluster/releases/latest/download/vcluster-darwin-arm64" && sudo install -c -m 0755 vcluster /usr/local/bin && rm -f vcluster  
  
curl -L -o vcluster "https://github.com/loft-sh/vcluster/releases/latest/download/vcluster-linux-amd64" && sudo install -c -m 0755 vcluster /usr/local/bin && rm -f vcluster  
  
curl -L -o vcluster "https://github.com/loft-sh/vcluster/releases/latest/download/vcluster-linux-arm64" && sudo install -c -m 0755 vcluster /usr/local/bin && rm -f vcluster

2\. **Deploy vcluster the GitOps-Way (Argo CD)**

In this part, I will show you how I deploy the vcluster distro k3s via Argo CD and the GitOps approach. First, vcluster-k3s is created as an application. We patch the necessary values directly via Argo CD.

![[Raw/Media/Resources/ee0eb5bdecd5973934e80ad904952f7f_MD5.png]]

Deploy vcluster k3s distro

You need only overwrite the values for the domain and the name of your cluster-issuer like:

apiVersion: argoproj.io/v1alpha1  
kind: Application  
metadata:  
  name: vcluster-k3s  
spec:  
  destination:  
    name: ''  
    namespace: vcluster-k3s  
    server: 'https://kubernetes.default.svc'  
  source:  
    path: helm/k3s  
    repoURL: 'https://github.com/la-cc/working-with-vcluster'  
    targetRevision: HEAD  
    helm:  
      parameters:  
        \- name: 'vcluster.syncer.extraArgs\[0\]'  
          value: '--tls-san=vcluster-k3s.example.com'  
        \- name: ingress.clusterIssuer  
          value: letsencrypt-dns  
        \- name: ingress.host  
          value: vcluster-k3s.example.com  
  sources: \[\]  
  project: default  
  syncPolicy:  
    automated:  
      prune: true  
      selfHeal: true

Then stage, commit and push the Application to your folder/repository that will be managed by Argo CD. Otherwise, you can apply the Application directly with *kubectl apply -f*.

3\. **Connect Argo CD to the vcluster**

In this step, we first connect (1) Argo CD (over the [cli](https://argo-cd.readthedocs.io/en/stable/cli_installation/)), which runs on the host cluster over the created ingress against the virtual cluster. After the successful connection, Argo CD creates a secret (2) in the *argocd* namespace. This contains the necessary information, such as the signed client cert, for Argo CD to connect.

![[Raw/Media/Resources/5597a234b8d16cc1d51056c2d6ba5c08_MD5.png]]

Connect Argo CD to the vcluster

You need only one line to create the connection.

vcluster connect vcluster-k3s -n vcluster-k3s --server=https://vcluster-k3s.example.com

Now you should see the following namespace:

kubectl get ns

  
NAME                     STATUS   AGE  
kube-public              Active   10h  
kube-node-lease          Active   10h  
default                  Active   10h  
kube-system              Active   10h

4\. **Deploy a simple application to the vcluster**

In this step (1) we are going to deploy a simple-webapp into the vcluster over Argo CD which is running on the host cluster. After the deployment, the service container called *syncer* will sync (2) the resource like ingress from the vcluster (guest cluster) to the host-cluster.

![[Raw/Media/Resources/d27801ee596e4a77e04be24cb9fbc2d3_MD5.png]]

You need only to overwrite the server with your vcluster server like:

apiVersion: argoproj.io/v1alpha1  
kind: Application  
metadata:  
  name: vcluster-simple-webapp  
spec:  
  project: bootstrap  
  source:  
    repoURL: 'https://github.com/la-cc/gitops-with-argocd'  
    path: helm-app  
    targetRevision: HEAD  
    helm:  
      releaseName: vcluster-simple-webapp  
      parameters:  
        \- name: ingress.enabled  
          value: 'true'  
        \- name: ingress.host  
          value: vcluster-simple-webapp.example.com  
  destination:  
    server: 'https://vcluster-k3s.example.com'  
    namespace: vcluster-simple-webapp  
  syncPolicy:  
    automated:  
      prune: true  
      selfHeal: true  
    syncOptions:  
      \- CreateNamespace=true

Now you should see a new namespace:

kubectl get  ns 

  
NAME                     STATUS   AGE  
kube-public              Active   10h  
kube-node-lease          Active   10h  
default                  Active   10h  
kube-system              Active   10h  
vcluster-simple-webapp   Active   8h

Get the ingress and see if you get a real IP like:

kubectl get ingress 

NAME                              CLASS   HOSTS                                              ADDRESS         PORTS     AGE  
vcluster\-simple\-webapp\-helm\-app   nginx   vcluster\-simple\-webapp.example.com                 20.22.13.15     80, 443   8h

If you access the ingress in your browser or your smartphone, you will get an answer from a real application with a real DNS hostname like:

![[Raw/Media/Resources/9a843fd6fe49b48733a5521ee4a78907_MD5.png]]

But how? How it looks like inside the host cluster? Change the context and get the namespaces.

kubectl get ns 

NAME              STATUS   AGE  
argocd            Active   3d5h  
calico\-system     Active   3d7h  
cert\-manager      Active   3d5h  
default           Active   3d7h  
external\-dns      Active   3d5h  
kube\-public       Active   3d7h  
kube\-system       Active   3d7h  
nginx\-ingress     Active   28h  
vcluster\-k3s      Active   10h

There is no vcluster-simple-webapp namespace in the host cluster. But how it works then? If you list all resources and the ingress in the namespace, I think you will understand.

kubectl get all,ing

NAME                                                                  READY   STATUS    RESTARTS   AGE  
pod/coredns-5486569497-hn8g7-x-kube-system-x-vcluster-k3s             1/1     Running   0          10h  
pod/vcluster-k3s-0                                                    2/2     Running   0          10h  
pod/vcluster-simple-webapp-helm-app-676475957f-ctbgr-x-v-5c0b42f37b   1/1     Running   0          8h

NAME                                                                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE  
service/kube-dns-x-kube-system-x-vcluster-k3s                             ClusterIP   10.128.0.150   <none>        53/UDP,53/TCP,9153/TCP   10h  
service/vcluster-k3s                                                      ClusterIP   10.128.0.244   <none>        443/TCP,10250/TCP        10h  
service/vcluster-k3s-headless                                             ClusterIP   None           <none>        443/TCP                  10h  
service/vcluster-simple-webapp-helm-app-x-vcluster-simple-we-02b7b53d73   ClusterIP   10.128.2.216   <none>        80/TCP                   8h

NAME                            READY   AGE  
statefulset.apps/vcluster-k3s   1/1     10h

NAME                                                                                        CLASS   HOSTS                                              ADDRESS         PORTS     AGE  
ingress.networking.k8s.io/vcluster-k3s-ingress                                              nginx   vcluster-k3s.valiant-dev.hpa-cloud.com             20.22.13.15     80, 443   10h  
ingress.networking.k8s.io/vcluster-simple-webapp-helm-app-x-vcluster-simple-we-02b7b53d73   nginx   vcluster-simple-webapp.valiant-dev.hpa-cloud.com   20.22.13.15     80, 443   8h

This is possible, because not only an emulation of kubernetes cluster is running in the host cluster, but also a service pod called *syncer*. This service will [sync](https://github.com/loft-sh/vcluster/blob/main/charts/k3s/values.yaml) resources like pods, services, ingress, etc. from the virtual guest cluster to the real host cluster.

## Use-Cases

There are numerous use cases, here are a few more that come to mind off the top of my head.

1.  **PR: Feature Test:** you can create a PR, this then triggers a pipeline, this creates a vcluster. It allows offering a self-service approach for the developers.
2.  **Cold Start for Kubernetes Newbies:** gives the teams a real environment to claim the first experiences with kubernetes.
3.  **Multi-Tenancy:** can be realized without any problems, with the difference to an isolated namespace approach. The developers can create namespaces, CRDs, etc. Resources are still controlled by quotas e for the namespace created by vcluster.
4.  **Multiple Version in Guest cluster:** it offers the possibility to try out different Kubernetes versions. This can be used to determine at an early stage whether there will be issues with the current configuration during an upgrade.

## Conclusion

In my opinion, it is amazing what the guys from lôft have created. I like the solution and the possibilities very well!

You get a real emulated kubernetes environment in an already existing Kubernetes environment. You can even deploy it using the GitOps approach. On top of that, you can connect the Argo CD instance from the host cluster against the virtual cluster and then deploy more applications in the virtual cluster again with the GitOps approach.

I am already looking forward to the larger k8s rootless mood solution. Thank you guys for the awesome solution!

## Contact Information

If you have some Questions, would like to have a friendly chat or just network to not miss any topics, then don’t use the comment function at medium, just feel free to add me to your [LinkedIn](https://www.linkedin.com/in/artem-lajko-%E2%98%81%EF%B8%8F-%E2%8E%88-82139918a/) network!

## References

-   vcluster — [https://github.com/loft-sh/vcluster](https://github.com/loft-sh/vcluster)
-   vcluster — [https://www.vcluster.com](https://www.vcluster.com/)
-   vcluster tls solution — [https://github.com/la-cc/working-with-vcluster](https://github.com/la-cc/working-with-vcluster)
-   helm example app — [https://github.com/la-cc/gitops-with-argocd/tree/main/helm-app](https://github.com/la-cc/gitops-with-argocd/tree/main/helm-app)