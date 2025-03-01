---
title: Enhancing Kubernetes Resource Management with SharedInformers | HackerNoon
source: https://hackernoon.com/enhancing-kubernetes-resource-management-with-sharedinformers
clipped: 2025-03-01
published: 
category: k8s
tags:
  - kubernetes
read: false
---

One of the roles of the Kubernetes controller is to monitor objects for the desired vs actual states and then send requests to change the existing state to the desired state at a controlled rate. The controller requests the [Kubernetes](https://hackernoon.com/419-stories-to-learn-about-kubernetes?ref=hackernoon.com) API server to retrieve an object’s information.

However, frequently retrieving data from the API server is costly and time-consuming. This is where Informer comes in, It is a client-side library that continuously watches for Kubernetes resources and keeps an in-memory copy of resources, which a given index can retrieve.

But how is it different from the raw watch command? Informers and watch both use resourceVersion to keep track of the objects. If you run a command `kubectl get pod <pod-name> -o yaml` you will notice a field called resourceVersion.

```
apiVersion: v1
kind: Pod
metadata:
  annotations:
    cni.projectcalico.org/podIP: 192.168.62.137/32
    kubernetes.io/psp: node-exporter
  creationTimestamp: "2021-12-12T14:35:11Z"
  labels:
    run: redis
  name: redis
  namespace: default
  resourceVersion: "39593432"
  uid: 233eaa4e-600f-4169-a617-538e5cbaa750
```

Informers use watch streams to get real-time updates. They relist all the objects periodically and store them in a cache while raw watch operations don't provide caching. You'd need to implement your own caching and event handling. Additionally, missing events are probable in the case of using just a raw watch.

For instance, we set up a watch operation to monitor Kubernetes jobs. If a network partition occurs, temporarily disconnecting our watch from the Kubernetes API server, we might miss critical events. During this disconnection, a job could be created and then quickly deleted. Once the network connection is restored, our watch operation resumes, but it has no way of knowing about the job that was briefly created and deleted during the outage. In this case, the informer would come in handy.

As various controllers may be watching over a particular resource and each controller has its informer, which maintains its cache, this could lead to synchronization issues. To solve this problem, it is usually recommended to use sharedInformers as they have a shared cache that multiple controllers can use. In SharedInformers, only one watch is opened on the Kubernetes API server regardless of the number of consumers, without adding extra overhead. The implementation of informers is present in the [client-go](https://pkg.go.dev/k8s.io/client-go/informers?ref=hackernoon.com) library. It is the official client library used internally by Kubernetes itself. Now let’s take a look at the components of SharedIndexInformer which is an instantiation of SharedInformer consisting of add and get Indexers ability (Indexer is a storage interface that lets you list objects using multiple indexing functions).

```
type SharedIndexInformer interface {
    SharedInformer
    
    AddIndexers(indexers Indexers) error
    GetIndexer() Indexer
}
```

The recommended way to create an informer is using [sharedInformerFactory](https://github.com/kubernetes/client-go/blob/3b969f96803febcac81f3a4272a9914270fbb3a3/informers/factory.go?ref=hackernoon.com#L55).

Below is an example code that will create and start a pod Informer.

```
factory := informers.NewSharedInformerFactory(client, 10*time.Minute)
podsInformer := factory.Core().V1().Pods().Informer()

stopCh := make(chan struct{})
factory.Start(stopCh) 
```

![[Raw/Media/Resources/7c0468a65b11fa22737726364a0ece63_MD5.gif]]

Components of Informer

### Reflector

The reflector consists of the ListerWatcher interface and thus opens a stream to check for events and proactively sends requests over regular intervals.

```


type ListerWatcher interface {
Lister
Watcher
}
```

This ensures the controller is not only edge (event) triggered but also level triggered, so if an event is missed, it doesn’t affect the environment. You can learn more about edge and level-triggered logic from James Bowes’s article - “[Level Triggering and Reconciliation in Kubernetes](https://hackernoon.com/level-triggering-and-reconciliation-in-kubernetes-1f17fe30333d?ref=hackernoon.com)”.

### Delta FIFO

The Reflector uses Delta FIFO queues to store information about state changes. It is the main channel between the Reflector and the Indexer. It contains the changed object and type of the change/Delta (Added, Updated, Deleted, Replaced, and Sync) and ensures no duplicates.

### Indexer

Indexer is a local storage for storing resource objects with its indexing feature; it helps reduce the number of API calls made to etcd.

The Indexer implements thread-safe operations, allowing concurrent reads and serialized writes.

### SharedProcessor

SharedProcessor distributes the information to all registered event handlers

**Finally, here is an overview of the function calls involved after invoking SharedInformer:**

![[Raw/Media/Resources/7c0468a65b11fa22737726364a0ece63_MD5.gif]]

Overview of the function calls involved after invoking SharedInformer

Feel free to reach out if you have any questions :)