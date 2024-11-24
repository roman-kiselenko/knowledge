---
title: The Easiest Way to Debug Kubernetes Workloads
source: https://martinheinz.dev/blog/49
clipped: 2023-09-07
published: 2021-05-17
category: tools
tags:
  - containers
  - node
  - pod
  - docker
read: false
---

Debugging containerized workloads and *Pods* is a daily task for every developer and DevOps engineer that works with Kubernetes. Oftentimes simple `kubectl logs` or `kubectl describe pod` is enough to find the culprit of some problem, but some issues are harder to hunt down. In those cases you might try to use `kubectl exec` but even that might not be enough as some containers such as *Distroless* don't even contain shell that you could SSH into. So what do we have left, if all of the above fails? ...

## There Might Just Be a Better Way...

Sometimes you need to grab a bigger hammer or just use more appropriate tool for the task at hand. In case of debugging workloads on Kubernetes, that appropriate tool would be `kubectl debug`, which is a new command added not too long ago (v1.18) that allows you to debug running pods. It injects special type of container called *EphemeralContainer* into problematic Pod allowing you to poke around and troubleshoot. This can be very useful for cases described in the intro or in any other situation where interactive debugging is preferable or more efficient. So, `kubectl debug` looks like the way to go, but to use it we will need *ephemeral containers*, so what exactly are these?

Ephemeral containers are a sub-resource in Pod similar to normal `containers`. Unlike regular containers though, ephemeral containers are not meant for building applications, but rather for inspecting them. We don't define them at the creation time of a Pod, rather we inject them using special API into running Pod to run troubleshooting commands and to inspect environment of the Pod. Apart from these differences, ephemeral containers also lack some of the fields of basic containers, such as `ports` or `resources`.

Why do we need them, though? Can't we just use basic containers? Well, you cannot add containers to Pod as they're supposed to be disposable (or in other words - deleted and recreated at any time), which can make it difficult to troubleshoot hard to reproduce bugs that require inspection of Pod. That's why ephemeral containers were added to API - they allow you to add container to an existing pod, making it easier to inspect running pods.

Considering that ephemeral containers are part of Pod spec which is core of Kubernetes, how is that you (probably) haven't heard about it, yet? The reason why these are mostly unknown feature is because ephemeral containers are in early *Alpha* stage, which means they're not enabled by default. Resources and features in this stage might undergo big changes or be removed entirely in future versions of Kubernetes. Therefore, to use them you have to explicitly enable them using *Feature Gates* in `kubelet`.

## Configuring Feature Gates

We already established that we want to try `kubectl debug` out, so how do we enable ephemeral containers feature gate? Well, it depends on your cluster setup. For example, if you're using `kubeadm` to spin up create clusters, then you can use following *ClusterConfiguration* to enable ephemeral containers:

```
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.20.2
apiServer:
  extraArgs:
    feature-gates: EphemeralContainers=true
```

In the following examples though, we will use *KinD (Kubernetes in Docker)* cluster for simplicity and testing purposes, which also allows us to specify Feature Gates that we want enabled. So, to create our playground cluster:

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
featureGates:
  EphemeralContainers: true
nodes:
- role: control-plane
```

With the cluster running, we should verify that it indeed worked. The simplest way to see if this configuration got applied, is to inspect Pod API which should now include the ephemeralContainers section alongside the usual Containers:

```
~ $ kubectl explain pod.spec.ephemeralContainers
KIND:     Pod
VERSION:  v1

RESOURCE: ephemeralContainers <[]Object>

DESCRIPTION:
     List of ephemeral containers run in this pod....
...
```

This confirms that we have it and therefore we can start using `kubectl debug`. So, let's start off with simple example:

```
~ $ kubectl run some-app --image=k8s.gcr.io/pause:3.1 --restart=Never
~ $ kubectl debug -it some-app --image=busybox --target=some-app
Defaulting debug container name to debugger-tfqvh.
If you don't see a command prompt, try pressing enter.
/ #

# From other terminal...
~ $ kubectl describe pod some-app
...
Containers:
  some-app:
    Container ID:   containerd://60cc537eee843cb38a1ba295baaa172db8344eea59de4d75311400436d4a5083
    Image:          k8s.gcr.io/pause:3.1
    Image ID:       k8s.gcr.io/pause@sha256:f78411e19d84a252e53bff71a4407a5686c46983a2c2eeed83929b888179acea
...
Ephemeral Containers:
  debugger-tfqvh:
    Container ID:   containerd://12efbbf2e46bb523ae0546b2369801b51a61e1367dda839ce0e02f0e5c1a49d6
    Image:          busybox
    Image ID:       docker.io/library/busybox@sha256:ce2360d5189a033012fbad1635e037be86f23b65cfd676b436d0931af390a2ac
    Port:           >none<
    Host Port:      >none<
    State:          Running
      Started:      Mon, 15 Mar 2021 20:33:51 +0100
    Ready:          False
    Restart Count:  0
    Environment:    >none<
    Mounts:         >none<
```

We first start a Pod called `some-app` just so we have something to *"debug"*. We then run `kubectl debug` against this Pod, specifying `busybox` as an image for the ephemeral container, as well as a target which is the original container. Additionally, we also include `-it` arguments so that we immediately attach to container and get a shell session.

In the above snippet you can also see that if we describe the Pod after running `kubectl debug` on it, then its description will include *Ephemeral Containers* section with values we specified as command options earlier.

## Process Namespace Sharing

`kubectl debug` is quite powerful tool, but sometimes adding another container to a Pod might not be enough to get relevant information about the application running in Pod's other container. This might be the case when container being troubleshot doesn't include necessary debugging tools or even shell. In such situation we can use *Process Sharing* to allow us to inspect Pod's original container using our injected ephemeral container.

One problem though with process sharing is that it cannot be applied to existing Pods, therefore we have to create a new one with `spec.shareProcessNamespace` set to `true` and inject an ephemeral container into it. Doing this would be quite cumbersome, especially if we have to debug multiple pods/containers or just perform this repeatedly. Luckily, `kubectl debug` can do this for us using `--share-processes` option:

```
~ $ kubectl run some-app --image=nginx --restart=Never
~ $ kubectl debug -it some-app --image=busybox --share-processes --copy-to=some-app-debug
Defaulting debug container name to debugger-tkwst.
If you don't see a command prompt, try pressing enter.
/ # ps ax
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    8 root      0:00 nginx: master process nginx -g daemon off;
   38 101       0:00 nginx: worker process
   39 root      0:00 sh
   46 root      0:00 ps ax

~ $ cat /proc/8/root/etc/nginx/conf.d/default.conf
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;
...
```

The above snippet shows that with process sharing we can see everything inside the other container in a Pod including its processes and files, which can definitely be very handy for debugging.

As you probably noticed, in addition to `--share-processes` we also included `--copy-to=new-pod-name` because - as was mentioned - we need to create a new pod whose name is specified by this flag. If we then list running pods from another terminal we will see the following:

```
# From other terminal:
~ $ kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
some-app         1/1     Running   0          23h
some-app-debug   2/2     Running   0          20s
```

And that's our new debug Pod along the original application Pod. It has 2 containers in comparison to the original one as it also includes the ephemeral container.

Also, if you want to at any point verify whether the process sharing is allowed in some Pod, then you can run:

```
~ $ kubectl get pod some-app-debug -o json  | jq .spec.shareProcessNamespace
true
```

## Putting It To Good Use

Now that we have the feature gate enabled and know how the command works, let's try to put it a good use and debug some application. Let's imagine the following scenario - we've got an application that's misbehaving and we need to troubleshoot networking related issues in its container. The application doesn't have necessary networking CLI tools which we could use. To solve this, we can use `kubectl debug` in a following way:

```
~ $ kubectl run distroless-python --image=martinheinz/distroless-python --restart=Never
~ $ kubectl exec -it distroless-python -- /bin/sh
# id
/bin/sh: 1: id: not found
# ls
/bin/sh: 2: ls: not found
# env
/bin/sh: 3: env: not found
#
...

kubectl debug -it distroless-python --image=praqma/network-multitool --target=distroless-python -- sh
Defaulting debug container name to debugger-rvtd4.
If you don't see a command prompt, try pressing enter.
/ # ping localhost
PING localhost(localhost (::1)) 56 data bytes
64 bytes from localhost (::1): icmp_seq=1 ttl=64 time=0.025 ms
64 bytes from localhost (::1): icmp_seq=2 ttl=64 time=0.044 ms
64 bytes from localhost (::1): icmp_seq=3 ttl=64 time=0.027 ms
```

After starting a pod we first try to get shell session into its container, which might seem like it worked, but when we try to run some basic commands we can see that there's literally nothing there. So, instead, we inject ephemeral container into the pod using `praqma/network-multitool` image which contains tools like `curl`, `ping`, `telnet`, etc. and now we can perform all the necessary troubleshooting.

In the above example it was enough for us to have another container in the Pod and poke around in there. But sometimes, you might need to look directly into the troubling container while not having a way to get into its shell. In that case we can take advantage of process sharing like so:

```
~ $ kubectl run distroless-python --image=martinheinz/distroless-python --restart=Never
~ $ kubectl debug -it distroless-python --image=busybox --share-processes --copy-to=distroless-python-debug
Defaulting debug container name to debugger-l692h.
If you don't see a command prompt, try pressing enter.
/ # ps ax
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    8 root      0:00 /usr/bin/python3.5 sleep.py  # Original container is just sleeping forever
   14 root      0:00 sh
   20 root      0:00 ps ax
/ # cat /proc/8/root/app/sleep.py
import time
print("sleeping for 1 hour")
time.sleep(3600)
```

Here we once again ran container that uses Distroless image. Knowing that we wouldn't be able to do anything in its shell, we ran `kubectl debug` with `--share-processes --copy-to=...`, which creates a new Pod with additional ephemeral container which has access to all processes. When we then list running processes, we can see that our application container's process has PID 8, which we can use to explore its files and environment. To do that, we have to go through `/proc/>PID</...` directory - which in this case would be - `/proc/8/root/app/...`.

Another common situation is that application keeps crashing upon container start making it difficult to debug as there's not enough time to get shell session into the container and run some troubleshooting commands. In this case the solution would be to create container with different entry point/command, which would stop the application from crashing immediately and allowing us to perform debugging:

```
~ $ kubectl get pods
NAME                READY   STATUS             RESTARTS   AGE
crashing-app        0/1     CrashLoopBackOff   1          8s

~ $ kubectl debug crashing-app -it --copy-to=crashing-app-debug --container=crashing-app -- sh
If you don't see a command prompt, try pressing enter.
# id
uid=0(root) gid=0(root) groups=0(root)
#
...

# From another terminal
~ $ kubectl get pods
NAME                READY   STATUS             RESTARTS   AGE
crashing-app        0/1     CrashLoopBackOff   3          2m7s
crashing-app-debug  1/1     Running            0          16s
```

## Bonus: Debugging Cluster Nodes

This article was mainly focused on debugging of Pods and their containers - but as any cluster admin knows - oftentimes it's the nodes that need debugging and not the Pods. Luckily for us, `kubectl debug` also allows for debugging of nodes by creating Pod that will run on specified node with node's root filesystem mounted in `/root` directory. This essentially acts as a SSH connection into node, considering that we can even use `chroot` to get access to host binaries:

```
~ $ kubectl get nodes
NAME                 STATUS   ROLES                  AGE   VERSION
kind-control-plane   Ready    control-plane,master   25h   v1.20.2

~ $ kubectl debug node/kind-control-plane -it --image=ubuntu
Creating debugging pod node-debugger-kind-control-plane-hvljt with container debugger on node kind-control-plane.
If you don't see a command prompt, try pressing enter.
root@kind-control-plane:/# chroot /host

# head kind/kubeadm.conf
apiServer:
  certSANs:
  - localhost
  - 127.0.0.1
  extraArgs:
    feature-gates: EphemeralContainers=true
    runtime-config: ""
apiVersion: kubeadm.k8s.io/v1beta2
clusterName: kind
controlPlaneEndpoint: kind-control-plane:6443
```

In the above snippet we first identified the node which we want to debug, then we ran `kubectl debug` explicitly using `node/...` as parameter to get access to our cluster's node. After that, when we get attached to the Pod, we use `chroot /host` to break out of `chroot` jail and gain full access to the host. Finally, to verify that we really can see everything on the host, we view part of `kubeadm.conf` in which we can see the `feature-gates: EphemeralContainers=true` which we configured in the beginning of the article.

## Conclusion

Being able to quickly and efficiently debug applications and services can save you a lot of time, but more importantly it can greatly help you with solving issues that might end-up costing you a lot of money if not resolved immediately. That's why it's important to have tools like `kubectl debug` at your disposal and enabled, even when they're not GA or enabled by default yet.

If - for whatever reason - enabling ephemeral containers is not an option, then it's probably a good idea to try practicing alternative debugging approaches, such as using debug version of application's image which would include troubleshooting tools; or temporarily changing Pod's container's command directive to stop it from crashing.

With that said, `kubectl debug` and ephemeral containers are only one of many useful - yet barely known - Kubernetes Feature Gates, so keep an eye out for followup article(s) that will dive into some other hidden features of Kubernetes.