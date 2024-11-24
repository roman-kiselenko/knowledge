---
title: 10 Things I wish I’d known before building a Kubernetes CRD controller
source: https://omerxx.com/k8s-controllers/
clipped: 2023-09-19
published: 2022-08-29
category: k8s
tags:
  - development
read: true
---

> Give me six hours to chop down a tree and I will spend the first four sharpening the axe.
> 
> -   A. Lincoln

Well I didn’t even know there was an axe… K8s resources, in that context, are one heck of a tree to chop. You better come ready to work. I hope that this information will find the axe for someone outthere and smooth out the process.

---

Let’s bring some order to the chaos. K8s documents the notions of [controllers](https://kubernetes.io/docs/concepts/architecture/controller/) and [operators](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/). The reader may be puzzled by the subtle differences between the two after reading them both.

An “Operator” is a pretty name coined by [CoreOS](https://web.archive.org/web/20170129131616/https://coreos.com/blog/introducing-operators.html) back in 2016, to describe the concept of managing application infrastructure using controllers and custom resources.

K8s Custom Resource Definitions (CRDs) allow users to extend the system with the same tools used to create and manage Pods, ReplicaSets, StatefulSets, and ConfigMaps. This notion is discussed in the “kubebuilder” book - [“Building a CronJob”](https://book.kubebuilder.io/cronjob-tutorial/cronjob-tutorial.html) tutorial. The creation of a `CronJob` component using an existing `Job` resource is an incredible example of an operator or controller. In a [2018 keynote in KubeCon](https://omerxx.com/k8s-controllers/*https://www.youtube.com/watch?v=AUNPLQVxvmw), Maciej Szulik, the creator of CronJob actually claimed its not yet implemented with a full on controller features.

Described in the K8s docs, a “controller” is a component that utilizes a control loop to bring the “desired state” of the system to its actual state. An operator is, in that sense, a controller with a CRD, and a story.

---

In a [K8s podcast episode](https://overcast.fm/+MqPlklnlw), Daniel Smith, Co-TL of SIG API, explains a powerful concept; “K8s is more like a database than an event-driven system”. He explains that instead of having different components communicating with one another, K8s holds information like a database. This information is the **state of the cluster**. An update or creation of a resource will result in a new **desired state**. Through a control loop, the controller takes care of achieving the desired state as events of its kind are received. Through iterations of the control loop, the desired state is reconciled with the existing state.

“Reconciliation” is noted here as the coined term for achieving [equilibrium](https://www.merriam-webster.com/dictionary/equilibrium). You’ll find the `Reconcile` loop in most controllers as the heart of the logic, and where events start their way.

> K8s is more like a database, then an event-driven system.

---

I wish someone had told me that. There are so many resources out there on how to build K8s operators and controllers, some like the Operator Framework automate things even further, helping the user construct operators based on native language, Helm, and others.

That said, [Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder), does most of this work for you already, and IMHO, better. It comes with a (really incredible) piece of documentation: [The Kubebuilder Book](https://book.kubebuilder.io/).

A must read, if you plan to build a K8s controller.

The book takes the reader from concept, through a real-life example and its step-by-step development process. It covers code generation, APIs, concepts of control loops and reconciliation, deployment and local development. Almost every piece of information needed for such a project.

The author has put a lot of care into writing the book and it’s easy to read. The concepts explored in the text are interesting and come with code snippets to underscore the points.

---

Now that’s a surprise, I had to double-check the data thoroughly to work out that `metaData` was never there. You can’t find it anywhere on the root-level and it’s seemed to be missing everywhere else.

My own case involves the creation of a `statefulSet` as part of my CRD, which I thought would be treated as a first-class citizen. On the contrary; nested meta objects will be ignored unless instructed specifically not to be. I don’t have answers to “why” (even though treated as a [bug](https://github.com/kubernetes-sigs/controller-tools/issues/448)), only the “how”:

In its [docs](https://book.kubebuilder.io/reference/controller-gen.html), the Code Gen CLI, sates the additional option to set `crd:generateEmbeddedObjectMeta=true` to allow nested meta objects. In the context of the makefile code generator, this would look something along the lines of:

```
.PHONY: manifests
manifests: controller-gen ## Generate WebhookConfiguration, ClusterRole and CustomResourceDefinition objects.
	$(CONTROLLER_GEN) rbac:roleName=manager-role \
                    crd:generateEmbeddedObjectMeta=true,maxDescLen=0 \
                    webhook paths="./..." \
                    output:crd:artifacts:config=config/crd/bases
```

You may also note the `maxDescLen=0`; During the development process (and afterwards), the amount of sheer text generated to every piece of the resource documentation is unbearable. Just try to `kubectl describe` your resource to find your terminal slowly losing its history. This doesn’t have to be “0”, but can help bring the lines of text to a manageable situation.

---

CRDs are great. You can create any *kind* (pun not intended) of object in K8s and manage it using a controller.

What about trying to interact with it outside of the controller’s context? You may be tempted to ask “why would anyone do that”. The way I see it:

1.  Customers may want to interact with this production service using their own controller or [informer](https://www.cncf.io/blog/2019/10/15/extend-kubernetes-via-a-shared-informer/).
2.  Depending on the system built, other pieces of software may need access to the object produced. Exactly this was the usecase I had to deal with - a Daemonset pod that updated data on new types of CRDs based on node-centered events.

It is relatively straight-forward to access all native resources with [client-go](https://github.com/kubernetes/client-go):

```
clientSet, _ := kubernetes.NewForConfig(config)
pods := clientSet.CoreV1().Pods("")
```

Custom resources, however, not only require you to obtain the relevant types for the object, once you do so, there is no client exported or generated for you. Additionally, once you finally reach the client you were looking for, you discover a full-on raw HTTP API with unstructured JSON requests and responses. Working this way isn’t fun (or safe).

### Discovery number one: “The dynamic client”[Permalink](#discovery-number-one-the-dynamic-client "Permalink")

The [dynamic client](https://github.com/kubernetes/client-go/tree/master/dynamic) for K8s is described well in this [blog post](https://caiorcferreira.github.io/post/the-kubernetes-dynamic-client/). The TL;DR is that it allows direct access to any kind of object within the cluster, whether structured or not. Here is one of the main components of the dynamic client that deals with `unstructured` objects:

> `unstructured.Unstructured`: This is a special type that encapsulates an arbitrary JSON while also complying with standard Kubernetes interfaces like `runtime.Object`
> 
> -   [The Kubernetes dynamic client](https://caiorcferreira.github.io/post/the-kubernetes-dynamic-client/)

Provided a `schema.GroupVersionResource` ([GVR](https://ddymko.medium.com/understanding-kubernetes-gvr-e7fb94093e88)), the dynamic client will fetch the CRD object and return it as an unstructured data object:

```
ctx := context.Background()
gvr := schema.GroupVersionResource{
  Group: "group.example.com",
  Version: "v1alpha1",
  Resource: "myresource",
}
returnedObj, err := c.Resource(gvr).
                    Namespace("default").
                    Get(ctx, "myresource-sample", metav1.GetOptions{})
if err != nil {
  return
}
```

It’s all fine and dandy, but what do you actually do with raw data if you need to do more than just parse it? According to the post mentioned above, you can modify nested data fields using functions like `unstructured.NestedInt64`, parsing and changing raw data in-place and returning it back to the cluster.

It felt like there was more to it:

### Discovery number two: “The DefaultUnstructuredConverter”[Permalink](#discovery-number-two-the-defaultunstructuredconverter "Permalink")

The `runtime` library exposes a converter function with a `FromUnstructured` method for unmarshaling data into the known CRD type. This creates typed, mutable, and accessible information from the unstructured data:

```
var myResource MyResourceType
err = runtime.
      DefaultUnstructuredConverter.
      FromUnstructured(returnedObj.UnstructuredContent(), &myResource)
if err != nil {
  return err
}
```

From fetching the object to parsing and manipulating it away from the context of the controller, that’s a complete interaction with a CRD.

Since my use-case involves customers, I wanted to provide them the option to interact with the generated objects, so I exposed it through my own client using a few simple methods that handled CRUD logic:

```
func Get(c dynamic.Interface, nsn types.NamespacedName) (MyResource, error)
func List(c dynamic.Interface, nsn types.NamespacedName) (MyResource, error)
func Update(c dynamic.Interface, mr MyResource) (error)
func Delete(c dynamic.Interface, mr MyResource) (error)
```

---

Finalizers are logic processes that are required before a K8s resource is deleted. The book says so, but it’s easy to skip. Perhaps you noticed a `deletionTimestamp` added to the metadata of an object when you tried to delete it. This is a finalizer that prevents garbage collection of the resource. For example, in AWS EBS, the internal logic backs the EBS controller attempts to delete the physical volumes before removing a PVC (removing the finalizer field in order to do so). As soon as a finalizer is removed, the object is automatically collected by the GC and removed from the cluster. With this method, you, the user, do not have to deal with waste and cluster leftovers.

---

There is another very important concept, yet easy-to-skip section in the Kubebuilder “[implementing a controller](https://book.kubebuilder.io/cronjob-tutorial/controller-implementation.html)”: The K8s garbage collector knows to remove these objects when the parent object is deleted, just as the CronJob creates Jobs. If your CRD generates other resources in the cluster, like a Pod, or Deployment, then you must set an owner reference.

When constructing the object, make sure to include:

```
# r being the reconciler receiver
if err := ctrl.SetControllerReference(parentObj, childObj, r.Scheme); err != nil {
  return nil, err
}
```

---

Changing them is quite the challenge, to say the least. Think about the group and resource names when you are creating the controller, CRD, API, etc. There are several reasons why these are important:

1.  Imports are made throughout the project. Imports must be a. unambiguous, b. straightforward, and c. named meaningfully
2.  They will be used in different places, such as yaml files holding the objects’ API Versions and Kinds. As well as in other areas such as the discussed GVR or GVK (GroupVersionKind) objects. If there is a meaningful group name other than “crd” or “apps”, use it. If not, keep in mind that it must make sense in the context of `<group-name>.company.com` as part of the `apiVersion` field.
3.  Changing them is a pain in the bum, especially with custom resource objects. Besides being part of every generated function or kubebuilder generator markers, it is probably mentioned hundreds of times throughout the code. Refactoring isn’t necessary, but not doing so will reduce the project’s readability. Simply put: Do not change the name of the CRD, unless you must. You’ve been warned 😉

---

That’s all there is to it. Please let me know if you have any other dos and don’ts to add or remove. In any case, that’s all the things I wish I knew instead of spending hours figuring them out for myself.

Thanks for reading!