---
title: The Birth Of a Kubernetes Pod — Deep dive to Kubernetes internals
source: https://sitereliability.in/deep-dive-the-birth-of-a-kubernetes-pod-understand-the-kubernetes-internals?s=35
clipped: 2023-11-23
published: 
category: k8s
tags:
  - pod
read: false
---

We will dive into the inner workings of Kubernetes, exploring each stage involved in bringing a Pod to life. Prepare to embark on a voyage filled with technical wonders!

Trust me it will be boring sometime, but we try to include as many in-depth details as possible so sit tight and enjoy :-)

***let's create a Kubernetes pod and see what the heck happens inside”………!***

### [Permalink](#heading-stage-1-pod-blueprinting "Permalink")**Stage 1: Pod Blueprinting:**

> *Defining the Pod Specifications!*

— Defining the Desired State The process commences with the creation of a Pod specification,  
typically in YAML format. This will include container image details, resource requirements, environment variables, volume mounts, networking configurations, and more. It acts as the blueprint for the Pod’s existence within the Kubernetes cluster.

### [Permalink](#heading-stage-2-kubectl-in-action "Permalink")**Stage 2: “Kubectl” In Action**

> *“Making the Connection: How Kubectl Sends Requests and Commands to the Kubernetes API Server”*

Here we’ll hit enter with the below command:
Here we are communicating with the Kubernetes API Server Using the Kubernetes command-line interface, such as “**kubectl**”,

Initially, kubectl executes ***client-side validation*** as its primary task. This validation guarantees that any requests destined to inevitably fail, such as attempts to create a resource that is not supported or employing an improperly formatted image name, are swiftly rejected before being sent to kube-apiserver. Let kubectl handle the silly mistakes, sparing the busy kube-apiserver’s valuable time

It utilizes generators to ***Assemble data into an HTTP request***, which is then sent to the kube-api server. Additionally, kubectl ***handles versioning negotiation***, ensuring compatibility between the client and server versions. When it comes to ***client authentication***, kubectl diligently searches for credential locations, prioritizing specified usernames. It includes authentication details, such as x509 certificates, bearer tokens, or basic authentication, in the HTTP request sent to the kube-api server. In the case of OpenID authentication, users are responsible for obtaining and including the token as a bearer token in the request. With these capabilities, kubectl seamlessly interacts with the Kubernetes ecosystem.

So, Here we submitted api request to kube-api servers lets see what next…

Stage 3: Behind the API Server Curtain:

> ***The API server*** *acts as the central control plane, receiving and processing requests from clients. It verifies the request, ensures proper authentication, and begins the orchestration process.*

**i) Authentication: Proving Our Identity**  
Each request runs through the authenticator chain until one succeeds. Whether it’s verifying TLS keys, checking bearer tokens, or validating basic auth credentials, kube-apiserver ensures that the requester’s identity is authenticated. Successful authentication removes the Authorization header and attaches user information to the request context, enabling subsequent steps like authorization and admission controllers to access the established user identity.

***ii) Authorization: Permissions Granted***  
Authentication grants us entry, but authorization determines whether we have the necessary permissions to perform the requested action. Kube-apiserver handles authorization by configuring a chain of authorizers,

\-Assemble a chain of Authorizers requests will go through it (Webhook , ABAC,RBAC,Node)  
— -> If all deny then the request fail with “Forbidden” — -> If any Authorizer Approve then It will proceed to next stage…

***iii) Admission Control: Upholding Cluster Rules***\*:\*  
An admission controller is a piece of code that intercepts requests to the Kubernetes API server prior to the persistence of the object, but after the request is authenticated and authorized.

Run through Admission controller chain, there are chains of plugins — -> If any one of the chains fails then the request will fail. ( Keep in mind: Expectation here is a little bit different than auth², Request should pass all Admission controller chains here)

Eg:- - Admission controller Plugins:  
1\. LimitRanger — set default container request and limit.  
2\. ResourceQuota — calculate and Dney request if the object counts in namespace is more than requested.(eg: pods,rc,service,load balancers)

I know you might have more doubts in mind regarding Adminssion controller (I’ll write a separate deep dive blog for specifically for adminssion controller.. stay tuned )

kube-apiserver deserializes the HTTP request, constructs runtime objects from them (kinda like the inverse process of kubectl’s generators), and persists them to the datastore.  
So to summarise: our Deployment resource now exists in etcd.

### [Permalink](#heading-stage-4-pod-house-hunting "Permalink")**Stage 4: Pod House-Hunting**

![](https://miro.medium.com/v2/resize:fit:996/1*8cDdXl2Eo1F2nNGTQ9MB-w.gif)

> ***Scheduling —*** *Finding a Suitable Home:*  
> *See How baby is doing — “Our pods, however, are stuck in a* `Pending` state because they have not yet been scheduled to a Node. The final controller that resolves this is the scheduler.

Scheduler will go through series of process to make the decision…

![](https://miro.medium.com/v2/resize:fit:2000/1*rTxPHqXonOqKW4XnMwKQ5Q.png)

[https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)

*Step 1: Node Selection* The scheduler begins by selecting a set of suitable nodes where the Pod can potentially be scheduled. It takes into account various factors such as resource requirements, node capacity, affinity, anti-affinity rules, and taints and tolerations. This ensures that the Pod lands on a node that can meet its resource demands and satisfies any specific placement constraints.

*Step 2: Filtering* Once a list of potential nodes is identified, the scheduler applies a series of filters to eliminate nodes that don’t meet certain criteria. These filters can be based on factors like node availability, node selectors, node conditions, resource limits, and more. Nodes that fail to pass these filters are removed from the candidate pool.

*Step 3: Scoring* After filtering, the scheduler assigns scores to the remaining nodes to prioritize their suitability for hosting the Pod. Each node receives a score based on factors such as resource availability, proximity to other Pods, inter-pod affinity or anti-affinity rules, quality of service requirements, and any custom rules or preferences set by the user. The higher the score, the more favorable the node is for scheduling.

*Step 4: Final Selection* With scores assigned, the scheduler performs a final evaluation and selects the node with the highest score as the chosen host for the Pod. If there are multiple nodes with the same highest score, the scheduler may employ additional tie-breaking mechanisms, such as random selection or user-defined rules, to determine the final winner.

*Step 5: Binding and Assignment* Once the node is selected, the scheduler informs the Kubernetes control plane about its decision. The selected node’s name is recorded in the Pod’s binding information, marking the assignment of the Pod to that particular node. The control plane updates the cluster’s state to reflect this assignment.

Stage 4: Node Allocation — Preparing the Host Once a node is selected, the Kubernetes controller responsible for managing the node begins the allocation process. It prepares the node by configuring necessary namespaces, network bridges, and resources required for the Pod’s execution. The container runtime, such as Docker or containerd, is then invoked to create the container within the Pod.

### [Permalink](#heading-stage-5-let-the-containers-party "Permalink")**Stage 5: Let the Containers Party**

> ***Here, the Container Creation Happens:***

![](https://miro.medium.com/v2/resize:fit:1400/1*OuXfcIUU-VShb3kXmEU2BA.png)

**How do Pods get IP?**

Introduction to the kubelet and its role in managing Pod lifecycles:  
The kubelet is an agent that operates on every node within a Kubernetes cluster. It plays a vital role in managing the lifecycle of Pods. Alongside other responsibilities, the kubelet translates the abstraction of a Pod into its constituent containers. It handles tasks such as managing container lifecycle, volume mounting, container logging, and garbage collection.

Pod synchronization process: The kubelet periodically queries the kube-apiserver, usually every 20 seconds, to obtain the list of Pods associated with the node it is running on. It compares this list against its own internal cache to detect new additions or discrepancies. If a Pod is being created, the kubelet registers startup metrics and generates a PodStatus object representing the Pod’s current Phase. The Pod’s Phase summarizes where it stands in its lifecycle, such as Pending, Running, Succeeded, Failed, or Unknown.

Admission handlers: After generating the PodStatus, the kubelet runs admission handlers to ensure the Pod has the correct security permissions. These handlers enforce security measures like AppArmor profiles and NO\_NEW\_PRIVS. If a Pod is denied at this stage, it remains in the Pending state until the security issues are resolved.

CRI and pause containers: The kubelet utilizes the Container Runtime Interface (CRI) to interact with the underlying container runtime, such as Docker or rkt. CRI provides an abstraction layer, allowing the kubelet to communicate with various runtime implementations. When starting a Pod, the kubelet invokes the RunPodSandbox remote procedure command, creating a “sandbox” that serves as a parent for the Pod’s containers. In the case of Docker, the sandbox creation involves setting up a “pause” container. This pause container hosts namespaces (IPC, network, PID) that are shared among the containers in the Pod.

CNI and pod networking: The kubelet delegates pod networking tasks to Container Network Interface (CNI) plugins. CNI enables different network providers to use various networking implementations for containers. The kubelet communicates with CNI plugins by streaming JSON data, configuring network settings. The bridge CNI plugin, for example, sets up a local Linux bridge in the host’s network namespace and connects it to the pause container’s network namespace using a veth pair. It assigns an IP address to the pause container’s interface and sets up routes, thus providing the Pod with its own IP address.

Inter-host networking: To facilitate communication between Pods on different hosts, Kubernetes often employs overlay networking. Overlay networks, like Flannel, dynamically synchronize routes across multiple nodes in a cluster. Flannel provides a layer-3 IPv4 network between nodes, encapsulating outgoing packets in UDP datagrams to ensure proper routing.

Container startup: Once networking is established, the kubelet proceeds with container startup. It pulls the required container image, using any secrets specified in the PodSpec for private registries. The kubelet then creates the container via CRI, populating a ContainerConfig struct with relevant details from the PodSpec. CPU manager assigns containers to sets of CPUs, and the container is subsequently started. Post-start container lifecycle hooks, such as Exec or HTTP actions, can be executed if registered.

> *\- And that, my friends, is how a pod comes to life!*
> 
> *It may not involve storks or delivery rooms, but it’s no less magical. So the next time you witness a pod taking its first breath in your cluster, imagine it wearing a tiny Kubernetes cap and entering the world with a burst of containerized energy. Birthdays are special, even for pods! Now, let’s raise a virtual toast to these little bundles of container joy as they embark on their exciting journey in the world of distributed systems.*  
> *Cheers to the birth of pods!!!!*
> 
> *(comment for missing points — I’ll try to include)*

\----------------------------------------------------------------------------------------

Thanks:  
Diagrams & Images are not created by me - All credit for that goes to the creators.

📚 "Stay in the know! Subscribe to the blog for the latest insights on tech trends, DevOps and Cloud strategies, and more. 📚#StayTuned  
**\- Site [Reliability.in](http://reliability.in/)**