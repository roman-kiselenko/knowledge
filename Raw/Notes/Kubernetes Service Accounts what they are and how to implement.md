---
title: "Kubernetes Service Accounts: what they are and how to implement"
source: https://medium.com/@jrkessl/kubernetes-service-accounts-what-they-are-and-how-to-implement-9b3701c667d0
clipped: 2023-11-03
published: 
category: k8s
tags:
  - development
read: true
---

Now you are trying to create, or maybe just to understand, what is a Kubernetes Service Account. Nice! It is in scope of this article:

1.  What is a Kubernetes Service Account
2.  How to create a Kubernetes Service Account
3.  How to use a Service Account so that your Pod can talk to the Kubernetes API (such as using the kubectl command)

To stand out from other Service Account articles around the web, I will, beyond just explaining what they are and showing you the create command, I will demonstrate, from scratch, how to setup a Pod with a Service Account and show you the difference to not using one.

![[Raw/Media/Resources/ad8f37616bdb361c1d9fd6cef2864fc0_MD5.jpg]]

Image made with Dall-E 2 and imgflip.com

Requirements: to fully understand this article you need to know the basics of a Kubernetes cluster: how to create and list namespaces, Pods, and Deployments. What are manifest files (.yaml files) and what are they good for.

A Service Account (SA) [provides an identity for a process that runs in a Pod](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/). Let me explain.

Usually a Pod just talks to other Pods. Your typical microservice running in a Pod just needs to receive external requests and talk to other Pods, maybe to an external database service. Your typical Pod does not talk to the Kubernetes API.

But when you interact with the cluster using the kubectl program, and you issue commands to list Pods, create Deployments and so on, you are instead talking to the Kubernetes API. You have your credentials, that may be saved in file ~/.kube/config, these credentials map to a user in the cluster, and this user has some permissions granted through the RBAC permissions model.

But consider you have a Pod that must interact with the cluster, that must use the Kubernetes API. Your Pod could require to communicate with the API for reasons such as:

1.  This Pod may be a CI/CD pipeline runner and from some Git repository it will deploy new cluster objects.
2.  You have some monitoring tool, and it needs to query Kubernetes itself about how much nodes, deployments, etc it is currently serving.

These Pods do not use credentials like you do. Instead, they use Service Accounts. These SAs allow these Pods to have an identity, and to this identity you can grant the appropriate permissions.

Service Accounts are namespaced objects. The create command is as simple as:

```
kubectl create sa -n mynamespace mysa
```

Now, describe it.

```
kubectl describe sa -n mynamespace mysa
```

Output:

```
Name:                mysa  
Namespace:           mynamespace  
Labels:              <none>  
Annotations:         <none>  
Image pull secrets:  <none>  
Mountable secrets:   <none>  
Tokens:              <none>  
Events:              <none>
```

## How about the Service Account Secret?

Until Kubernetes 1.23, when you created a Service Account it automatically created a Secret object with a token. It would look like this:

```
Name:                mysa  
Namespace:           mynamespace  
Labels:              <none>  
Annotations:         <none>  
Image pull secrets:  <none>  
Mountable secrets:   mysa-token-xsr67  
Tokens:              mysa-token-xsr67  
Events:              <none>
```

Inside this Secret object there is a token. With this token a process running outside the cluster could get authenticated against the Kubernetes API.

From Kubernetes 1.24 onwards, creating a Service Account no longer automatically creates the Secret with a Token. [From Kubernetes 1.22 this type of Secret is no longer used to mount credentials into Pods, and if you need this token, obtaining tokens via the TokenRequest API is recommended](https://kubernetes.io/docs/concepts/configuration/secret/#service-account-token-secrets) instead of using Service Account token Secret objects. The demonstration you will read below does not require this Secret object and it’s token.

Let’s put that theory in practice and it will all make much more sense. This example I have done in an AWS EKS Kubernetes cluster. The cluster version is 1.24.8. Consider your Pod will talk to the Kubernetes API using the kubectl program. Let’s start creating this.

First let’s get us a container image with just that: the kubectl program. Check this Dockerfile:

```dockerfile
FROM alpine  
ARG KUBECTL\_VERSION=1.24.8  
  
RUN apk add --update --no-cache curl ca-certificates bash git  
  
RUN curl -sLO https://storage.googleapis.com/kubernetes-release/release/v${KUBECTL\_VERSION}/bin/linux/amd64/kubectl && \\  
    mv kubectl /usr/bin/kubectl && \\  
    chmod +x /usr/bin/kubectl  
  
ENTRYPOINT \[ "sleep", "36000" \]
```

You may think this Dockerfile is stupid. And you may be correct. But this is not important. What’s important is that this is just what we need now: a container that has just kubectl.

Let’s deploy this container in a Pod in our cluster. First we build and publish the container.

```
docker build . -t jrkessl/sleeperk8s:1.24.8  
docker push jrkessl/sleeperk8s:1.24.8
```

If you are following along, you don’t need to build and publish the container yourself. You can just use mine: *jrkessl/sleeperk8s:1.24.8*. I plan to leave it online on docker hub.

To deploy it as a container Pod, I use this file, deployment.yml:

```
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: dummypod  
  namespace: mynamespace  
spec:  
  selector:   
    matchLabels:   
      app: dummypod  
  replicas: 1  
  template:  
    metadata:  
      labels:  
        app: dummypod  
    spec:  
      containers:  
        \- image: jrkessl/sleeperk8s:1.24.8  
          name: vs
```

And apply with:

```
kubectl create ns mynamespace  
kubectl apply -f deployment.yml
```

Now we have a Pod with kubectl. Let’s 1) get the Pod name 2) enter the Pod and 3) try to retrieve namespaces:

```
  
$ kubectl get pods -n mynamespace  
NAME                        READY   STATUS    RESTARTS   AGE  
dummypod-85f998fcdb-vqw4t   1/1     Running   0          11m
```

  
```
$ kubectl exec -it -n mynamespace dummypod-85f998fcdb-vqw4t -- bash   
dummypod-85f998fcdb-vqw4t:/

  
dummypod-85f998fcdb-vqw4t:/  
Error from server (Forbidden): namespaces is forbidden: User "system:serviceaccount:mynamespace:default" cannot list resource "namespaces" in API group "" at the cluster scope
```

See the response. We see that kubectl is present in the Pod and it runs fine. We also see that it already knows how to reach the Kubernetes API of the cluster it resides (we don’t need *aws eks update-kubeconfig*). We went ahead and tried to retrieve namespaces, but kubectl responded: *“system:serviceaccount:mynamespace:default” cannot retrieve namespaces.* But what is *“system:serviceaccount:mynamespace:default”*? It is a Service Account. It is called *default* and it exists in *mynamespace* namespace. This is the default SA.

## Every Pod has the default Service Account, at least.

Every Pod has a SA. It will be the default one from the namespace in question if you don’t specify it. Try this command below.

```
kubectl get pod/<the name of your pod> -n mynamespace -o yaml | grep -i serviceaccount
```

It will retrieve the source code of the Pod we just deployed and will filter it for the occurrence of the term “serviceaccount”, case insensitive.

```
kubectl get pod/dummypod-85f998fcdb-kq8fb -n mynamespace -o yaml | grep -i serviceaccount  
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount  
  serviceAccount: default  
  serviceAccountName: default  
      - serviceAccountToken:
```

We found 2 keys in this Pod’s manifest. “serviceAccount” and “serviceAccountName”.

Let’s try changing the SA of this Pod. If you get this Pod’s definition, change the SA, and reapply, it works. But we have created a Deployment, not a Pod. So let’s do it on the Deployment.

## How to specify a Service Account in a Deployment

If you retrieve the yaml of your Deployment with *get deployment… -o yaml* you won’t find the SA because it’s not an explicitly mandatory specification. Create a Deployment without specifying the SA and Kubernetes will assume the default action, which is to use the default SA of whichever namespace you are using. I will show now how and where to specify the SA for a deployment. I will skim through this topic because it’s an adjacent topic and I don’t want to lose focus. But check this out:

We already know we are working with the Deployment object, but if you wanted to see the objects supported by your current cluster:

```
kubectl api-resources
```

Then you can extract the complete definition of Deployments into a file:

```
kubectl explain Deployment - recursive > file.txt
```

In this file, search for serviceaccount and you will find these 2 keys:

```
spec.template.spec.serviceAccount  
spec.template.spec.serviceAccountName
```

Explain them with:

```
kubectl explain deployment.spec.template.spec.serviceAccount  
kubectl explain deployment.spec.template.spec.serviceAccountName
```

With the explain command you will see that the first one is deprecated, and the second one is just what we are looking for. Let’s apply it to our Deployment object. See the last line, it’s the only one changed.

```
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: dummypod  
  namespace: mynamespace  
spec:  
  selector:   
    matchLabels:   
      app: dummypod  
  replicas: 1  
  template:  
    metadata:  
      labels:  
        app: dummypod  
    spec:  
      containers:  
        \- image: jrkessl/sleeperk8s:1.24.8  
          name: vs  
      serviceAccountName: mysa
```

Now apply the Deployment to your cluster again with *kubectl apply -f deployment.yml* and, from within the Pod, try to use kubectl again to retrieve namespaces:

  
```
$ kubectl apply -f deployment.yml  
deployment.apps/dummypod configured
```

  
```
$ k exec -it dummypod-55bb96689d-m7qnl -n mynamespace -- kubectl get ns  
Error from server (Forbidden): namespaces is forbidden: User "system:serviceaccount:mynamespace:mysa" cannot list resource "namespaces" in API group "" at the cluster scope  
command terminated with exit code 1
```

Notice how it no longe says *mynamespace:default* can’t retrieve namespaces. Because it’s now using the SA we created: *mynamespace:mysa*. Next we need to grant Service Account *mynamespace:mysa* the privilege we want it to have.

If you have read [my RBAC article](https://medium.com/@jrkessl/kubernetes-kbac-permissions-model-and-how-to-add-users-to-aws-eks-c6d642f79a6d) you should know this command will easily grant the SA admin privileges in the cluster. Admin privileges are too much for a production system. But it’s OK for this learning exercise.

```
$ kubectl create clusterrolebinding mycrb \\  
\> --clusterrole=cluster-admin \\  
\> --serviceaccount=mynamespace:mysa
```

Now try again to retrieve namespaces from within the Pod and it will work!

```
$ kubectl exec -it dummypod-55bb96689d-m7qnl -n mynamespace -- kubectl get ns  
NAME              STATUS   AGE  
default           Active   32m  
kube-node-lease   Active   32m  
kube-public       Active   32m  
kube-system       Active   32m  
mynamespace       Active   24m
```

Done! Reviewing what we did and saw here:

1.  We created namespace *mynamespace* and Service Account *mysa* in it
2.  We created a Deployment and assigned *mysa* to it
3.  We created a clusterrolebinding object granting *mysa* admin privileges in the cluster so that the Pod can use kubectl command to talk to the Kubernetes API.
4.  We seen how Service Account Secret objects with their tokens are no longer automatically created from Kubernetes 1.24 onwards, but they aren’t really needed anyway, not for the use case we did, from 1.22 onward.

Did you like this article? Do you think the style, wording, or granularity level of the explanations could be improved, so you had understood more, from the first time you read it? Let me know, I would like your opinion.