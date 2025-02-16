---
title: Kubernetes Pod IP Conflict
source: https://hwchiu.medium.com/kubernetes-pod-ip-conflict-7b3c6c63ef4f
clipped: 2023-09-24
published: 
category: k8s
tags:
  - pod
  - k8s
read: false
---

I recently encountered an interesting case where newly created Pods were assigned the same IP address as existing Pods. This led to a conflict and disrupted the normal accessibility of services within the cluster.

Although this situation is not very common, when it does occur, it can be challenging to identify the problem since Kubernetes itself is not responsible for IP allocation. Instead, it relies on the Container Network Interface (CNI) to handle this task, with the Kubelet acting as the intermediary between CNI and Kubernetes.

In this blog post, we will simulate an environment to explore this issue and clarify its nature. Once all the mysteries are unraveled, we will examine the potential scenarios in real-world situations where this problem might arise.

To simulate the environment, we will use KIND to create a multi-node Kubernetes cluster. The version and configuration files are as follows:

```shell
→ kind \--version  
kind version 0.9.0
```

```
→ cat kind.yaml  
kind: Cluster  
apiVersion: kind.x-k8s.io/v1alpha4  
nodes:  
\- role: control-plane  
  image: kindest/node:v1.18.8@sha256:f4bcc97a0ad6e7abaf3f643d890add7efe6ee4ab90baeb374b4f41a4c95567eb  
\- role: worker  
  image: kindest/node:v1.18.8@sha256:f4bcc97a0ad6e7abaf3f643d890add7efe6ee4ab90baeb374b4f41a4c95567eb  
\- role: worker  
  image: kindest/node:v1.18.8@sha256:f4bcc97a0ad6e7abaf3f643d890add7efe6ee4ab90baeb374b4f41a4c95567eb  
\- role: worker  
  image: kindest/node:v1.18.8@sha256:f4bcc97a0ad6e7abaf3f643d890add7efe6ee4ab90baeb374b4f41a4c95567eb

→ kubectl version  
Client Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.4", GitCommit:"d360454c9bcd1634cf4cc52d1867af5491dc9c5f", GitTreeState:"clean", BuildDate:"2020-11-11T13:17:17Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"linux/amd64"}  
Server Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.8", GitCommit:"9f2892aab98fe339f3bd70e3c470144299398ace", GitTreeState:"clean", BuildDate:"2020-09-14T07:44:34Z", GoVersion:"go1.13.15", Compiler:"gc", Platform:"linux/amd64"}

→ kubectl get nodes  
NAME                 STATUS   ROLES    AGE   VERSION  
kind-control-plane   Ready    master   18m   v1.18.8  
kind-worker          Ready    <none\>   17m   v1.18.8  
kind-worker2         Ready    <none\>   17m   v1.18.8  
kind-worker3         Ready    <none\>   17m   v1.18.8
```

We will simulate a large number of Pods using a deployment. The content of the deployment is as follows:

```yaml
→ cat debug.yaml  
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: debug-pod  
spec:  
  replicas: 30  
  selector:  
    matchLabels:  
      app: debug-pod  
  template:  
    metadata:  
      labels:  
        app: debug-pod  
    spec:  
      containers:  
        \- name: debug-pod  
          image: hwchiu/netutils:latest 
```

Deploying Pods To simulate the issue, we will deploy Pods.

1.  Deploy Pods
2.  Stop the kubelet on the node and delete specific directories on the node
3.  Restart the kubelet on the node and deploy more Pods to that node

Next, we will proceed with the above steps, focusing on the node kind-worker3 as the node where the issue occurs.

## Deploy Pods

→ kubectl apply -f debug.yaml  
deployment.apps/debug\-pod created

Here, we will focus on observing the IP addresses of all Pods on kind-worker3.

 → kubectl get pods -o wide | grep kind-worker3  
debug-pod-7f9c756577-6nv7b   1/1     Running   0          3m38s   10.244.2.6    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-9l766   1/1     Running   0          3m38s   10.244.2.8    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-bg89r   1/1     Running   0          3m39s   10.244.2.9    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-hb2cr   1/1     Running   0          3m38s   10.244.2.7    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-kzkjr   1/1     Running   0          3m39s   10.244.2.3    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-ncqg8   1/1     Running   0          3m38s   10.244.2.5    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-pwk9v   1/1     Running   0          3m39s   10.244.2.2    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-twwpn   1/1     Running   0          3m39s   10.244.2.4    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-z7kmq   1/1     Running   0          3m39s   10.244.2.11   kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-zgs7k   1/1     Running   0          3m38s   10.244.2.10   kind-worker3   <none\>           <none\>

Currently, there are a total of 10 Pods on kind-worker3, and all their IP addresses are different. This is completely normal.

## Stop the kubelet on the node and delete specific directories on the node

○ → sudo docker exec kind-worker3 systemctl stop kubelet  
○ → sudo docker exec kind-worker3 systemctl status kubelet  
● kubelet.service - kubelet: The Kubernetes Node Agent  
     Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: enabled)  
...

After confirming the stoppage, proceed to delete the relevant directories.

sudo docker exec kind-worker3 rm -rf /run/cni-ipam-state

## Restart the kubelet on the node and deploy more Pods to that node

After completing the above steps, let’s restart the kubelet.

○ → sudo docker exec kind-worker3 systemctl start kubelet  
○ → sudo docker exec kind-worker3 systemctl status kubelet  
● kubelet.service - kubelet: The Kubernetes Node Agent  
     Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: enabled)  
    Drop-In: /etc/systemd/system/kubelet.service.d  
             └─10\-kubeadm.conf  
     Active: active (running) (thawing) since Fri 2020\-11\-27 04:04:28 UTC; 3s ago  
       Docs: http://kubernetes.io/docs/

Next, we will deploy new Pods onto kind-worker3. The approach I’m taking is to utilize the concept of draining, where we remove the Pods from kind-worker and kind-worker2 and then redeploy them onto kind-worker3. We will observe the IP addresses of these Pods during this process.

○ → kubectl drain kind-worker \--ignore-daemonsets  
○ → kubectl drain kind-worker2 \--ignore-daemonsets

→ kubectl get pods -o wide | grep kind-worker3 | sort -k 6  
debug-pod-7f9c756577-hkhx9   1/1     Running   0          54s   10.244.2.10   kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-zgs7k   1/1     Running   0          11m   10.244.2.10   kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-qmjqk   1/1     Running   0          54s   10.244.2.11   kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-z7kmq   1/1     Running   0          11m   10.244.2.11   kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-pd6hl   1/1     Running   0          54s   10.244.2.12   kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-ssmsv   1/1     Running   0          54s   10.244.2.13   kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-bbx6j   1/1     Running   0          54s   10.244.2.14   kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-9rn4j   1/1     Running   0          54s   10.244.2.15   kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-sfwhs   1/1     Running   0          54s   10.244.2.16   kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-jc8nw   1/1     Running   0          54s   10.244.2.17   kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-5sb28   1/1     Running   0          54s   10.244.2.18   kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-mgtzx   1/1     Running   0          54s   10.244.2.19   kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-64xlq   1/1     Running   0          53s   10.244.2.20   kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-jpmfk   1/1     Running   0          53s   10.244.2.21   kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-pwk9v   1/1     Running   0          11m   10.244.2.2    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-rhfk4   1/1     Running   0          71s   10.244.2.2    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-kzkjr   1/1     Running   0          11m   10.244.2.3    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-njmw6   1/1     Running   0          71s   10.244.2.3    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-l9fl8   1/1     Running   0          71s   10.244.2.4    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-twwpn   1/1     Running   0          11m   10.244.2.4    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-ncqg8   1/1     Running   0          11m   10.244.2.5    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-rshx5   1/1     Running   0          71s   10.244.2.5    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-6nv7b   1/1     Running   0          11m   10.244.2.6    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-vklgs   1/1     Running   0          71s   10.244.2.6    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-hb2cr   1/1     Running   0          11m   10.244.2.7    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-jkpbd   1/1     Running   0          54s   10.244.2.7    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-9l766   1/1     Running   0          11m   10.244.2.8    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-lr8t5   1/1     Running   0          54s   10.244.2.8    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-bg89r   1/1     Running   0          11m   10.244.2.9    kind-worker3   <none\>           <none\>  
debug-pod-7f9c756577-wx4s6   1/1     Running   0          54s   10.244.2.9    kind-worker3   <none\>           <none\>

Based on the results you mentioned, we can observe that the IP address 10.244.2.11 and the previous addresses (.2 to .10) have all been duplicated. Up to this point, we have created a scenario of duplicated IP addresses through a peculiar operational process. However, there are still two questions remaining:

1.  Why does this process lead to IP address duplication?
2.  While this process seems clever, what real-world scenarios might result in a similar situation?

To answer these questions, let’s delve into the underlying causes and explore potential scenarios that can result in IP address conflicts.

From this point onwards, let’s delve into the essence of this issue. Once we understand the overall workflow of Kubernetes, we will have a better understanding of how to approach and narrow down the occurrence of this problem.

Before we begin, we need to have a basic concept of how IP addresses are obtained.

In reality, Kubernetes itself does not directly handle the allocation of IP addresses. There are only two things it does:

1.  The controller sets certain values, such as NodeCIDR, on the nodes based on configuration parameters (not discussed in this article).
2.  When the Kubelet creates a Pod, it ultimately calls the Container Network Interface (CNI) to handle the network setup.

Next, let’s take a look at the flow of the CNI process using a diagram (a simplified flow, as the actual underlying calls may vary slightly).

Initially, the Kubelet creates a sandbox for the Pod, at which point the Pod does not have any IP address.

![[Raw/Notes/Raw/Media/Resources/7d34648b7e5e7e2fa22b8bf999f6298a_MD5.jpg]]

The CNI is responsible for providing networking capabilities to the Pod, including IP address allocation. By default, the CNI configuration files are stored in /etc/cni/net.d.

The Kubelet reads /etc/cni/net.d to determine which CNI is being used on the system.

![[Raw/Notes/Raw/Media/Resources/4b3421c30ae3d94f1b3a39ab4f319b38_MD5.jpg]]

→ docker exec kind-worker3 cat /etc/cni/net.d/10-kindnet.conflist

{  
        "cniVersion": "0.3.1",  
        "name": "kindnet",  
        "plugins": \[  
        {  
                "type": "ptp",  
                "ipMasq": false,  
                "ipam": {  
                        "type": "host-local",  
                        "dataDir": "/run/cni-ipam-state",  
                        "routes": \[  
                                {  
                                        "dst": "0.0.0.0/0"  
                                }  
                        \],  
                        "ranges": \[  
                        \[  
                                {  
                                        "subnet": "10.244.2.0/24"  
                                }  
                        \]  
                \]  
                }  
                ,  
                "mtu": 1500

        },  
        {  
                "type": "portmap",  
                "capabilities": {  
                        "portMappings": true  
                }  
        }  
        \]  
}

The example above shows the CNI configuration file used within KIND. Pay attention to the IPAM field, which specifies the use of the “host-local” service and stores the data in “/run/cni-ipam-state”.

Thus, when the Kubelet finds this configuration, it calls the “kind-net” CNI to handle the task. “kind-net” then utilizes “host-local” to handle the IP address allocation issue.

![[Raw/Notes/Raw/Media/Resources/2bed30dc67ec567c93af9f2f2ace9a09_MD5.jpg]]

Call host-local IPAM to handle IP

“host-local” uses the local file “/run/cni-ipam-state/kindnet” as its local database to track which IP addresses have been used. It searches for an available IP address from the database and returns the result to the “kind-net” CNI for further processing.

![[Raw/Notes/Raw/Media/Resources/4547aaf6aaccb0089758e402eb453e4e_MD5.jpg]]

host-local use local file as cache

○ → docker exec kind-worker3 ls /run/cni-ipam-state/kindnet  
10.244.2.10  
10.244.2.11  
10.244.2.12  
10.244.2.13  
10.244.2.14  
10.244.2.15  
10.244.2.16  
10.244.2.17  
10.244.2.18  
10.244.2.19  
10.244.2.2  
10.244.2.20  
10.244.2.21  
10.244.2.3  
10.244.2.4  
10.244.2.5  
10.244.2.6  
10.244.2.7  
10.244.2.8  
10.244.2.9  
last\_reserved\_ip.0  
lock

We can see that this section records the current status of all CNIs in use. It’s important to note that **not every CNI/IPAM uses the “host-local” approach**. As the name suggests, “host-local” operates independently on each node and relies on a folder with multiple files to track the assigned IP addresses.

Once everything is complete, “kind-net” sets the relevant IP addresses on the Pods.

![[Raw/Notes/Raw/Media/Resources/97b9dc24722ada859bad6d19c43cc38e_MD5.jpg]]

Assign IP to Pod

## Let’s review again.

Going back to the simulated environment, it becomes clear how the entire process unfolds. Initially, we deployed a large number of Pods on kind-worker3, and “host-local” started writing files for each assigned IP address and recording them in “/run/cni-ipam-state/kindnet”.

Next, we stopped the kubelet, temporarily preventing the CNI from being invoked, and removed all the states maintained by “host-local”.

At this stage, the system becomes inconsistent. The CNI’s state information for IP address management is removed, but the containers using those IP addresses are still active.

Finally, by redeploying Pods onto kind-worker3, “host-local” attempts to generate IP addresses again. Since “/run/cni-ipam-state/kindnet” is empty, it starts the calculation anew, resulting in IP addresses being assigned that were previously used, leading to IP conflicts.

Having reached this point, we now have a solid understanding of the issue. However, do we encounter such problems in real-world scenarios?

To meet the conditions for this problem to occur, we need a few factors:

1.  The chosen CNI ultimately uses “host-local” as the local database for recording the used Pod IP addresses.
2.  The folder used by “host-local” on the system is deleted, causing all its data to be lost, without “host-local” being aware of it.

Satisfying condition one is relatively simple, as Flannel, for example, uses this approach by default to track IP addresses.

As for condition two, unless unexpected modifications occur to system files, we usually do not encounter this issue.

However, there is a specific use case where both conditions are met. It involves installing Kubernetes using Rancher and using Flannel as the CNI. When restoring Kubernetes using Rancher’s restore feature, we encounter this problem.

During the restore process in Rancher, the kubelet is shut down first, and the node undergoes some cleanup operations, such as the removal of certain directories.

const (  
 ToCleanEtcdDir          = "/var/lib/etcd/"  
 ToCleanSSLDir           = "/etc/kubernetes/"  
 ToCleanCNIConf          = "/etc/cni/"  
 ToCleanCNIBin           = "/opt/cni/"  
 ToCleanCNILib           = "/var/lib/cni/"  
 ToCleanCalicoRun        = "/var/run/calico/"  
...  
)

func (h \*Host) CleanUpAll(ctx context.Context, cleanerImage string, prsMap map\[string\]v3.PrivateRegistry, externalEtcd bool) error {  
 log.Infof(ctx, "\[hosts\] Cleaning up host \[%s\]", h.Address)  
 toCleanPaths := \[\]string{  
  path.Join(h.PrefixPath, ToCleanSSLDir),  
  ToCleanCNIConf,  
  ToCleanCNIBin,  
  ToCleanCalicoRun,  
  path.Join(h.PrefixPath, ToCleanTempCertPath),  
  path.Join(h.PrefixPath, ToCleanCNILib),  
 }

 if !externalEtcd {  
  toCleanPaths = append(toCleanPaths, path.Join(h.PrefixPath, ToCleanEtcdDir))  
 }  
 return h.CleanUp(ctx, toCleanPaths, cleanerImage, prsMap)  
}

This function removes the ToCleanCNILib path, which points to /var/lib/cni/. Interestingly, the default path for “host-local” is also /var/lib/cni.

var defaultDataDir = "/var/lib/cni/networks"

  
  
type Store struct {  
 \*FileLock  
 dataDir string  
}

When these two factors coincide, it leads to IP conflicts.

From both perspectives, neither CNI/IPAM nor the containers using the IP addresses are necessarily at fault. They are simply performing their respective duties. The issue arises when the file responsible for maintaining the IP state for CNI/IPAM is removed, while the containers using those IP addresses remain active.

One approach to resolving this problem is to restart all the Pods on the affected node, allowing the CNI to start afresh. Alternatively, when installing Flannel using Rancher, modifying the configuration file to use a different location for IPAM (host-local) can also address the issue.

It’s important to note that Rancher (prior to version 2.5) is built on the concept of using etcd for backup and restoration of the entire Kubernetes cluster. Therefore, it is reasonable to perform node data cleanup during this process.

However, it’s important to acknowledge that etcd data alone does not fully represent the entire Kubernetes cluster, as there are other components that are not stored in etcd. For example, the CNI configuration mentioned in this article or other resources like PV/PVC. Achieving a complete backup and restoration of a Kubernetes cluster is almost impossible, not only due to technical limitations but also because it raises philosophical questions regarding how one defines their desired backup and restoration approach.