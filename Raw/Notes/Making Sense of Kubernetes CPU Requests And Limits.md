---
title: Making Sense of Kubernetes CPU Requests And Limits
source: https://medium.com/@jettycloud/making-sense-of-kubernetes-cpu-requests-and-limits-390bbb5b7c92
clipped: 2023-09-08
published: 
category: k8s
tags:
  - network
  - resources
read: false
---

![[Raw/Media/Resources/465aaa7a6720eaa2e1790b6d96d3ce26_MD5.png]]

Armen Shakhbazian is a Software Engineer at JettyCloud. Our company takes part in the development of an American UCaaS solution RingCentral.

For the last year Armen’s team has been moving services of RingCentral customers to Kubernetes and, as many other people going through the same process, had to deal with resource management: how to properly specify requests and limits for CPU and RAM to get good resource utilization and avoid performance issues.

In this article Armen will examine what «requests» and «limits» mean, how they translate to OS primitives and how they are enforced. In conclusion, he’ll give some useful metrics to monitor as well as a couple of recommendations on how to calculate requests and limits for your apps. Each section of this article has links to other articles and papers, so you can deeper explore given topics.

It will be beneficial for readers to have prior experience with Kubernetes and Linux, as some fundamentals are covered very briefly, just to establish common vocabulary. Furthermore, in this article Armen will focus on CPU, since other resources are relatively straight-forward.

Before we begin, a couple words about the project I worked on while writing this article. Our department at JettyCloud develops back-end services for RingCentral’s collaboration tool. For the last couple of years we have been moving backend services and infrastructure to GitOps and Kubernetes, and so far we deployed several clusters, hundreds of nodes and a couple thousands of pods.

At this scale, under-utilization of server resources can be costly, so we are very much motivated to not over-provision our infrastructure. On the other hand, we deal with a pretty diverse zoo of services performing a variety of tasks. Carelessly allocated resources can lead to poor user experience and, in some cases, cascade failures.

Kubernetes allows specifying how much CPU/RAM a single Pod needs and how to restrict usage of these resources for a given Pod. This is done via Requests and Limits under the Resources section.

```yaml
apiVersion: v1  
kind: Pod  
metadata:  
  name: my-app  
spec:  
  containers:  
  \- name: app  
    image: my.private.registry/my-app  
    resources:  
      requests:  
        memory: "64M"  
        cpu: "250m"  
      limits:  
        memory: "128M"  
        cpu: "500m"
```

Before examining how requests and limits are enforced, let’s get familiar with units they are measured in. The example above specifies 250 millicores and 64 megabytes as requests for container application and 500 millicores/120 megabytes as limits.

## Memory Units

Memory is measured in bytes, and units are pretty straight-forward. Kubernetes allows SI suffixes like k, M, G, T for kilobyte, Megabyte, Gigabyte and Terabyte respectively. As well as Ki, Mi, Gi, Ti for kibibyte, Mebibyte, Gibibyte and Tebibyte — which is a power of 2 units.

## CPU Units

CPU resources are measured, you guessed it, in CPU units. 1 CPU unit is equivalent to 1 physical (or virtual, depending on where cluster runs) core. To specify fractions of CPU, you can use millicore units, where 1 CPU = 1000m.

Since CPU is a compressible resource, it is very unintuitive what this unit corresponds to, and, to make things more confusing, the meaning of requests and limits are slightly different.

> **CPU is an absolute unit**, meaning that 1 CPU is the same no matter how many cores a Node has.

## Requests and Scheduling

Kubernetes uses the requests section of resources to schedule Pods on Nodes and guarantee that the Pod will receive the requested amount of resources.

Actually, resources are specified for each container in Pod, but for the sake of simplicity we assume that our Pods have only one container. Kubernetes uses total value for a Pod when scheduling of Pods on Nodes happens.

For example, let’s assume we have the following 3 Pods:

-   App 1: 250m CPU and 512M of Memory
-   App 2: 300m CPU and 512M of Memory
-   App 3: 350m CPU and 768M of Memory

When Kubernetes attempts to schedule Pods on a Node with 1 vCPU and 2G RAM, all pods will fit, since there is enough resources for them:

![[Raw/Media/Resources/bcef59c7ec62eba950a0b5c212a7eb9c_MD5.png]]

But what if we change the request for App 3, so that now it requires 1G of memory? Now Kubernetes will not be able to schedule that Pod to Node since there is not enough resources:

![[Raw/Media/Resources/27f3f069b6bfafcc25b5c9c5b62d33fd_MD5.png]]

This is pretty simple, and with RAM it is obvious how memory is distributed to Pods according to their requests. But what about the CPU? What does 250m even mean? Welp, let’s find out and take a look into the abyss of Linux CPU scheduler.

## CPU Requests, CPU Shares and CFS

To allocate resources to containers of Pod, Kubernetes uses **Сgroups and CFS** (completely fair scheduler) on Linux. Roughly speaking, all processes/threads of a container run in a distinct Cgroup, and CFS allocates CPU resources to these Cgroups according to specified resource requests (bear with me, it will be clear… eventually).

However, CFS operates with something called **CPU Shares**, and not CPU Units of Kubernetes. To convert CPU Units to CPU Shares, Kubernetes equates 1 CPU to 1024 CPU shares. Meaning that a Pod with 500m CPU requests will be given 512 CPU Shares. And a Node with 6 CPUs has 6144 CPU Shares total.

![[Raw/Media/Resources/5de2d32aa1d2768a37b7f0577f143e91_MD5.png]]

… ok. **But what does CPU Shares mean?** What does it mean for a Pod to have 512 shares? Left out of any context, it means nothing. Shares are a relative unit used by CFS to allocate CPU resources in time of contention.

When Pods do nothing or very little work and CPU is mostly idle, CFS does not care how many shares each Cgroup has. But when multiple Cgroups have runnable tasks, and there are not enough CPU resources, CFS makes sure each Cgroup get CPU time relative to how many shares they have. And, since Kubernetes computes shares from CPU units, it guarantees that a Pod receives requested CPU resources.

…>\_<. **But what are «CPU resources»?** How much CPU is 500m or 512 shares?

So, not going too deep into CFS internals, it works like this: CFS runs a task (thread) for a short time. When the task is interrupted (or a scheduler tick happens) the task’s CPU usage is accounted for: the (short) time it just spent using the CPU core is added to its CPU usage.

Once CPU usage for that task gets high enough so that another task becomes the least run, it picks that new task to schedule and the current task is pre-empted.

For example, 4 Pods, each with one container and single-threaded app, on a Node with 2 vCPU:

![[Raw/Media/Resources/41fb76660b9239ccbc45aa39be06db34_MD5.png]]

1.5ms is the minimum scheduler period in which a single task will run (/proc/sys/kernel/sched\_min\_granularity\_ns) — e.g. task is given at least 1.5 ms to run, but can use less. And remember that shares make sense only in time of contention, so we assume all apps are Runnable for all intervals.

For representation’s sake, it is better to operate with bigger periods, like 100ms or 1s.

With this in mind, lets define CPU requests:

> A CPU requests unit can be understood as the percentage of a given CPU period that is guaranteed to Pod.

E.g.: a 750m CPU for a period of 100 ms means that a POD is guaranteed to have 75 ms of CPU time each 100 ms (on one or multiple cores). A 5000m CPU for a period of 1s means that a Pod is guaranteed to have 5 s of CPU each 1 s (1 s on each of 5 cores).

It does not mean that if POD uses less resources than it requested, the CPU will remain idle. If another Pod is runnable at that time, CFS will schedule that Pod.

**That logic of scheduling can be extrapolated for multithreaded apps, as they are in most cases.** To reiterate, each POD container gets its own Cgroup, and threads/processes of the container are tasks in that Cgroup.

CFS ensures each Cgroup gets CPU time according to its shares, and each task within that Cgroup also gets enough CPU time.

![[Raw/Media/Resources/300a179a1f0710d830625cf287db8d59_MD5.png]]

## Summary

Kubernetes guarantees that each pod gets CPU/RAM according to its requests. **If Kubernetes does not have enough resources on Nodes, it will fail to schedule Pod.**

Memory resources are straightforward and are defined in bytes, Kubernetes makes sure that a Node has enough memory to provide for scheduled Pods according to their requests.

To calculate CPU requests, Kubernetes converts CPU units to CPU shares, **where 1 CPU = 1024 shares.** It then ensures that each Node has enough shares to guarantee CPU time for each POD.

These CPU shares have meaning only in relation to total shares on a Node and are used by linux CFS to distribute CPU resources across Pods.

**To make sense of CPU units, one can think of them as a percentage of a given CPU period that is guaranteed to POD**. E.g.: 150 m CPU for a 100 ms period means that a Pod is guaranteed to have 15 ms CPU time each 100 ms period. 4 CPU for 1 s period means that Pod is guaranteed to have 4 s of CPU time each 1 s period = 1 s on each of 4 CPU cores.

## Links

-   [CPU Shares for Kubernetes](https://www.batey.info/cgroup-cpu-shares-for-kubernetes.html)
-   [Kubernetes CPU Requests Explained](https://mechpen.github.io/posts/2020-07-25-k8s-cpu/index.html)
-   [Understanding Linux Container Scheduling](https://engineering.squarespace.com/blog/2017/understanding-linux-container-scheduling)
-   [Kernel.org: CFS Design](https://www.kernel.org/doc/Documentation/scheduler/sched-design-CFS.txt)
-   [Towards Achieving Fairness in Linux Scheduler](https://www.cs.mtsu.edu/~waderholdt/6450/papers/cfs.pdf)
-   [Linux Scheduler: A Decade of Wasted Cores](https://people.ece.ubc.ca/sasha/papers/eurosys16-final29.pdf)

Kubernetes uses limits to restrict max resource consumption for a Pod. For memory, it is pretty simple: **if an application attempts to allocate more memory than specified in Limits — it will be killed by OOMKiller.**

CPU Limits, on the other hand, are enforced via throttling. **When an application attempts to use more CPU than it is limited to — CFS will throttle it.**

## CFS Periods and Quotas

As for requests, CFS operates on a level of Cgroups. For each Cgroup limits are defined by two configurable parameters:

-   **cpu.cfs\_quota\_us** — CPU time in microseconds available to cgroup in a period, computed from Limits value.
-   **cpu.cfs\_period\_us** — accounting period in microseconds after which allocatable resources are refilled, by default is 100 ms.

**To compute the quota, Kubernetes equates 1 CPU to 1 full period.** E.g.: if cfs\_period\_us is 100ms, then 1CPU is 100ms, 2500m CPU is 250ms and 750m CPU is 75ms.

For various performance and efficiency reasons, CFS tracks quota usage for each Cgroup and each core. In other words, each Cgroup has a pool of allocatable CPU resources equal to cfs\_quota\_us and refilled in cfs\_period\_us. Within Cgroup each CPU also has a pool of resources, and it is filled from Cgroup pool in 5 ms slices (default of sched\_cfs\_bandwidth\_slice\_us).

![[Raw/Media/Resources/4cd630bff802501c1873af284739a231_MD5.png]]

Do not confuse slice size with how much time a thread actually runs! 5 ms is the size of a slice in which the CPU pool is refilled. But threads can consume only fractions of that time, e.g. sub-millisecond amount. To properly distribute quota across threads, unused resources from the CPU pool returned to the Cgroup pool. **It’s also worth remembering that threads are throttled only when they are in a Runnable state** — idle or waiting threads will not be scheduled and not be throttled.

> Now we can define what a CPU Limit is**: CPU Limit can be understood as the percentage of each scheduler CPU period (100ms) that Pod cannot exceed.**

E.g.: The limit of 750 m CPU means that each 100 ms period a Pod cannot use more than 75 ms of CPU time. 2.5 CPU means that each 100ms period a Pod can use only 250 ms of CPU time (think multi-core system).

## More Cases of Throttling

**Setting insufficient CPU limits can cause unexpected throttling.** In most cases, it affects latency, specifically tail latency.

There are many factors that affect throttling probability aside from limits:

-   Overall system utilization (or under-utilization).
-   Number of threads in Cgroup (= in container = in Pod).
-   Incoming load (e.g. request rate).
-   IO latency.

Not to mention tunables like cpu.cfs\_period\_us / sched\_cfs\_bandwidth\_slice\_us and in-app configuration (something like GC params).

Here is an example when, depending on how the scheduler assigns threads, a request can be throttled and have additional latency or fit in and be processed without issues.

![[Raw/Media/Resources/c8ac5f5a5d06eeb812f69fca80c652c1_MD5.png]]

Another case is bursts in CPU usage, which is more prominent in apps with a large amount of threads. For example, let’s take a look at Pod with Node.JS application. In cluster mode and with some native modules it can spawn over 50 threads: v8 and libuv threads for master and worker processes, rd-kafka threads for each worker etc. In some unfortunate cases when most of these threads have work to do, quota can be quickly eaten, and that Pod will be throttled — hence p95 latency will spike.

There are less obvious reasons for throttling stemming from CFS implementation:

-   When returning unused resources from the CPU pool to Cgroup pool, CFS can reserve 1ms. This seems to be a tiny amount, but with numbers of cores and fast threads it can shove a noticeable amount from quota.
-   Since throttling is computed on a per-CPU pool, CFS can throttle threads a bit even when overall Cgroup quota is available.
-   There are still open issues in the Kubernetes repository related to unexpected throttling.
-   Some solutions, like burstable bandwidth control, cannot be configured from Kubernetes yet.

**First, one shall understand that throttling by itself is not an issue**, it should be avoided only when it causes problems in application behavior (think spikes in tail latency). For example, assume we have a service that does some background job or batch processing. If this service gets throttled from time to time it will most likely be fine: no latency for clients, no guarantees broken and so on.

On the other hand, if we have a service that handles incoming requests from clients and returns response «synchronously», throttling can affect the client experience — it may cause high latency spikes which will be perceived by clients as slowdown or glitches.

Now, let’s assume we’ve monitored our service, saw some latency degradation and CPU throttling, and we are determined to eliminate said throttling. Here is what can be done:

## 1\. Adjust or Remove CPU Limits

There are many articles on the internets suggesting that removing CPU limits are bad idea, but most of them give unsatisfying or wrong reasons:

**«Without limits, the container can use all the CPU resources available on the Node» — this is incorrect**.

Firstly, as described in previous sections, the scheduler will always do its best to provide each Pod a requested amount of CPU time. So, if CPU requests are set correctly, there is no chance that some glitchy Pod will consume all CPU.

Secondly, Kubernetes reserves some amount of CPU time for the system and its services (kube-proxy, etc.). So the Node will never end up in an unresponsive situation. Things may slow down, but will remain functioning.

**«Without setting limits equal to requests, you cannot get Guaranteed QoS class» — this is correct.**

However, Guaranteed QoS classes have no advantages from CPU perspective, it is only for incompressible resources like Memory. Over-provisioning of CPU requests does not worth eliminating the slight chance of Pod being killed during RAM contention. It is much better to set sensible Memory Limits.

Some valid reasons to keep CPU Limits:

1.  **Actually limiting CPU usage.** If you provide a service to run 3rd-party code, or bill by resource usage, using Kubernetes to enforce limits might be an option.
2.  **Predictable behavior.** For example, multiple latency-sensitive services can be scheduled on the same Node and work fine because there are free CPUs and pods can easily exceed their requests. After some deployment, configuration of Pods on that Node may change and suddenly there is not enough free CPU, scheduler does not allow Pods to exceed their requests and unexpected degradation occurs. Limits make such cases more obvious, since we can observe throttling in case of unexpected CPU demands.
3.  **Testing.** CPU Limits allow understanding how much of a CPU service actually needs and make proper calculation for production.
4.  **Predictable CPU utilization.** There are some reasons to desire CPU utilization less or equal to 70–80%, including queueing theory, CPU cache pollution and networking internals (but I’m out of my depth here and this is mostly my guesses).

In general, it is safe to set the CPU limit x2 or x3 of CPU requests, as long as you carefully calculate these requests.

## 2\. Control Threading in Application

Let’s again take Node.JS for example. Although the mantra goes like: «Node.JS is single-threaded», it refers to JavaScript code execution. Node.JS apps have libuv thread pool for some I/O and sync task processing, v8 threads for JavaScript processing and GC. Moreover, native libraries like node-rdkafka will spawn its own thread pools, and finally cluster mode will have master/worker processes with their separate threads. All in all, Pods with Node.JS app can have over 50 threads.

For golang apps, you might want to configure the GOMAXPROCS variable, since by default it calculated from the number of logical CPUs.

The same goes for Java services: there are threads for GC, thread pools for networking and for database connections.

By default, many thread pools (in Node.JS/Java/Go/etc) calculate their size depending on the number of CPU cores. Despite being in a container with limited CPU quota, processes will observe all cores on a Node.

All of these threads share the same quota and will create unnecessary contention and context switches, which may lead to throttling. It is recommended to adjust thread-pools in application for containerized execution.

## 3\. Adjust CFS tunables on Nodes

Default cpu.cfs\_period\_us of 100ms can be too high, and lowering these values can have a positive effect on throttling. There are some recommendations, for example [from zolando](https://www.slideshare.net/try_except_/optimizing-kubernetes-resource-requestslimits-for-costefficiency-and-latency-highload) to experiment with these knobs. As well as [low latency tuning guide](https://rigtorp.se/low-latency-guide/) for Linux kernel. This should be done with performance testing and careful understanding. Generally, previous options should be enough to eliminate throttling in most cases.

**Kubernetes uses limits to restrict maximum resource consumption of a Pod.** If a Pod uses more Memory than its configured limit — that Pod gets OOMKilled. If a Pod uses more CPU than its configured limit, it gets throttled.

CPU limit unit can be understood as the percentage of each scheduler CPU period (100 ms) that a Pod cannot exceed. E.g.: limit of 750 m CPU means that each 100 ms period POD cannot use more than 75 ms of CPU time.

Throttling may occur for various reasons, many of them can be obscure and require deep understanding of scheduler, application as well as Node state.

-   [How to Use CPU Management Policies Reasonably to Improve Container Performance](https://www.alibabacloud.com/blog/how-to-use-cpu-management-policies-reasonably-to-improve-container-performance_599140?spm=a2c65.11461447.0.0.7dbe38229yL1oS)
-   [The container throttling problem](https://danluu.com/cgroup-throttling/)
-   [Unthrottled: How a Valid Fix Becomes a Regression](https://engineering.indeedblog.com/blog/2019/12/cpu-throttling-regression-fix/)
-   [HN thread on CPU Limits](https://news.ycombinator.com/item?id=24351566)
-   [JVM GC and CFS Throttling](https://engineering.linkedin.com/blog/2016/11/application-pauses-when-running-jvm-inside-linux-control-groups)
-   [The burstable CFS bandwidth controller](https://lwn.net/Articles/844976/)
-   [Kernel.org: CFS Bandwidth control](https://docs.kernel.org/scheduler/sched-bwc.html)
-   [CPU Bandwidth Control on CFS](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/36669.pdf)
-   [Mitigating Unnecessary Throttling in Linux CFS Bandwidth Control](https://folk.ntnu.no/rakeshk/pubs/SBACPAD22.pdf)

Monitoring of a Pod resource usage is critical to estimate and adjust resource requests and limits. It is also important to do it properly, peeking at the right metrics to track.

## Memory

**Absolute memory usage per Pod:** simply monitor container\_memory\_working\_set\_bytes value.

```
sum(container\_memory\_working\_set\_bytes{cluster\_name="$cluster", namespace="$namespace", pod\_name=~"^$app\-.\*", container\_name!="POD"}) by (pod\_name)
```

**Relative to Requests:** ratio of container\_memory\_working\_set\_bytes / kube\_pod\_container\_resource\_requests\_memory\_bytes

```
sum(container\_memory\_working\_set\_bytes{cluster\_name="$cluster", namespace="$namespace", pod\_name=~"^$app\-.\*", container\_name!="POD"}) by (pod\_name) / sum(kube\_pod\_container\_resource\_requests\_memory\_bytes{cluster\_name="$cluster", namespace="$namespace", pod\_name=~"^$app\-.\*", container\_name!="POD"}) by (pod\_name)) > 0
```

**Relative to Limits:** ratio of container\_memory\_working\_set\_bytes / kube\_pod\_container\_resource\_limits\_memory\_bytes
```
sum(container\_memory\_working\_set\_bytes{cluster\_name="$cluster", namespace="$namespace", pod\_name=~"^$app\-.\*", container\_name!="POD"}\[1s\]) by (pod\_name) / sum(kube\_pod\_container\_resource\_limits\_memory\_bytes{cluster\_name="$cluster", namespace="$namespace", pod\_name=~"^$app\-.\*", container\_name!="POD"}) by (pod\_name)) > 0
```

## CPU

As noted previously, CPU requests/limits make sense as a percentage of some period. Given that, to monitor CPU usage we can use the increase of container\_cpu\_usage\_seconds\_total in 1s as a good representation of the actually used CPU time.

Usage in CPU Units: irate(container\_cpu\_usage\_seconds\_total)

```
sum(irate(container\_cpu\_usage\_seconds\_total{cluster\_name="$cluster", namespace="$namespace", pod\_name=~"^$app\-.\*", container\_name!="POD"}\[1s\])) by (node, pod\_name)
```

Usage in Percentage on Node: irate(container\_cpu\_usage\_seconds\_total) / kube\_node\_status\_allocatable\_cpu\_cores — will show familiar % of CPU usage on node.

```
sum(irate(container\_cpu\_usage\_seconds\_total{cluster\_name="$cluster", node=~"$node", container\_name!="POD", pod\_name=~"^$app\-.\*"}\[1s\])) by (node) / on(node) group\_left() kube\_node\_status\_allocatable\_cpu\_cores{cluster\_name="$cluster", node=~"$node"} OR on() vector(0)
```

Relative to Requests: irate(container\_cpu\_usage\_seconds\_total) / kube\_pod\_container\_resource\_requests\_cpu\_cores

```
sum(irate(container\_cpu\_usage\_seconds\_total{cluster\_name="$cluster", namespace="$namespace", pod\_name=~"^$app\-.\*", container\_name!="POD"}\[1s\])) by (pod\_name) / sum(kube\_pod\_container\_resource\_requests\_cpu\_cores{cluster\_name="$cluster", namespace="$namespace", pod\_name=~"^$app\-.\*", container\_name!="POD"}) by (pod\_name)) > 0
```

Relative to Limits: irate(container\_cpu\_usage\_seconds\_total) / kube\_pod\_container\_resource\_limits\_cpu\_cores

```
sum(irate(container\_cpu\_usage\_seconds\_total{cluster\_name="$cluster", namespace="$namespace", pod\_name=~"^$app\-.\*", container\_name!="POD"}\[1s\])) by (pod\_name) / sum(kube\_pod\_container\_resource\_limits\_cpu\_cores{cluster\_name="$cluster", namespace="$namespace", pod\_name=~"^$app\-.\*", container\_name!="POD"}) by (pod\_name)) > 0
```

Throttling: container\_cpu\_cfs\_throttled\_periods\_total / container\_cpu\_cfs\_periods\_total

```
sum(rate(container\_cpu\_cfs\_throttled\_periods\_total{cluster\_name="$cluster", namespace="$namespace", pod\_name=~"^$app\-.\*"} / container\_cpu\_cfs\_periods\_total{cluster\_name="$cluster", namespace="$namespace", pod\_name=~"^$app\-.\*"})) by (pod\_name)
```

Values of throttling are collected from cgroup/cpu.stat:

-   nr\_periods = container\_cpu\_cfs\_periods\_total — number of elapsed bandwidth control periods (cpu.cfs\_period\_us) when Cgroup tasks were runnable
-   nr\_throttled = container\_cpu\_cfs\_throttled\_periods\_total — number of periods in which tasks of Cgroup were throttled
-   throttled\_time = container\_cpu\_cfs\_throttled\_seconds\_total — sum total amount of time individual threads within the Cgroup were throttled

These metrics are a good foundation for application observability. It is recommended to monitor these values and adjust Requests/Limits if necessary.

I hope that after reading this article you have a better understanding of CPU resources in Kubernetes, how Requests are computed and balanced and how Limits are enforced.

As a wrap up, here are some recommendations for you to have smooth sailing with kubernetes:

**1\. Understand the threading model of your app and adjust it for the containerized environment.**

**For Node.JS apps:** do not use cluster modules as it creates unnecessary threads. Experiment with the number of libuv and v8 threads.

**For JVM apps:** tune thread-pool and GC settings.

**For Golang apps**: tune GOMAXPROCS or simply use [automaxprocs](https://github.com/uber-go/automaxprocs) package from Uber.

**2\. Measure CPU usage, set enough CPU Requests to handle prod traffic at peak time.**

Do performance testing, measure CPU usage and set CPU requests accordingly to handle peak time traffic. A good starting point is to have peak time CPU usage around 70–80% of CPU Requests. Do not over-provision to account for all spikes — that is what CPU Limits for.

**3\. Set CPU Limits for applications based on Requests.**

For latency-sensitive applications — think handling client requests — set Limits to be x2-x4 of Requests. Note that Grafana dashboards do not have enough granularity, and you will miss some spikes of CPU usage, so it is recommended to observe CPU throttling values and minimize them.

For background applications — think cron jobs or async event handling — set reasonable Limits x1.5-x2 of Requests. Requests/Limits combination should be such to comfortably handle peak time traffic and account for some extra load.

**4\. Add more metrics to apps, measure as much as you can and make it visible on monitoring dashboards.**

Metrics collected with node-exporter and such are not granular enough and may miss spikes of load/latency.

Collect in-app metrics correctly utilizing sliding-windows, interpolation and other techniques to make spikes visible. Don’t rely on averages, make tail-latencies visible — collect at least p50, p75, p99 quantiles. If Incoming requests severely vary in size/processing time — make sure they are collected into separate buckets or counters, so they don’t mask outliers and spikes.