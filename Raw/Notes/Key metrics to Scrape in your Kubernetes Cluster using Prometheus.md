---
title: Key metrics to Scrape in your Kubernetes Cluster using Prometheus
source: https://medium.com/@onai.rotich/key-metrics-to-scrape-for-effective-monitoring-and-optimization-of-a-kubernetes-cluster-c2dd80568a2e
clipped: 2023-09-04
published: 
category: observability
tags:
  - k8s
  - observability
read: false
---

![[Raw/Media/Resources/1a88489190d281be08bf7fdbf7a5659b_MD5.png]]

> When working with Kubernetes, monitoring and analyzing metrics is crucial for gaining visibility into the health and performance of the cluster. However, not all metrics are created equal, and it’s important to identify and focus on the most relevant ones.
> 
> This blog post will discuss the key metrics that should be scraped in Kubernetes to ensure effective monitoring and optimization of the cluster.

**Determine important metrics**

1.  Four Golden Signal
2.  USE Method
3.  RED method

**Sources of metrics**

1.  Node
2.  Kubelet and containers
3.  Kubernetes API
4.  etcd
5.  Derived metrics (kube-state-metrics)

**Metrics aggregation through the Kubernetes Hierarchy**

![[Raw/Media/Resources/fb020a8f9ce03952bc860c507c689dda_MD5.png]]

By reflecting on the most foundational aspects of your service, the four golden signals are the basic building blocks of an effective monitoring strategy. They improve the time to detect (TTD) and the time to resolve (TTR).

The four golden signals of monitoring are:

-   **Latency:** the time it takes to serve a request.
-   **Traffic:** the total number of requests across the network.
-   **Errors:** the number of requests that fail.
-   **Saturation:** the load on your network and servers.

If you can only measure four metrics of your user-facing system, focus on these four.

## Latency

Let’s say you have a web application that serves requests from users. Each time a user sends a request, the system needs to process that request and send back a response. The time it takes for the system to respond to that request is known as latency.

Now, let’s say that some of these requests are successful, while others fail due to errors. For example, a user might send a request to purchase an item, but the system encounters an error when attempting to process the payment, resulting in a failed request.

If we were to measure the overall latency of all requests, including both successful and failed ones, we would get a misleading picture of the system’s performance. The failed requests would skew the results and make it difficult to identify any underlying issues.

Instead, we need to differentiate between the latency of successful requests and the latency of failed requests. By doing so, we can identify any performance issues that are affecting the system and take steps to optimize it.

For example, if we notice that the latency of failed requests is consistently high, we can investigate the underlying cause of those errors and take steps to address them. This might involve optimizing the system’s code, improving the database performance, or upgrading the infrastructure to handle higher levels of traffic.

Overall, monitoring latency is essential for maintaining a healthy and performant system. By tracking the latency of successful and failed requests separately and setting targets for good latency rates, we can optimize the system and improve the overall user experience.

## Traffic

Let’s say you have a web service that provides an online shopping platform. As more and more customers use the platform to browse products, add items to their cart, and complete purchases, the demand placed on the system increases. This increased demand is known as traffic.

To measure traffic, we might use a high-level system-specific metric such as HTTP requests per second. This measures the number of requests being sent to the system each second. For example, if the web service is receiving 1000 HTTP requests per second, we know that the system is experiencing a high level of traffic.

We might also break down the traffic measurement further by looking at the nature of the requests. For example, we might distinguish between static content requests (such as loading a product image) and dynamic content requests (such as adding an item to the cart). By doing so, we can identify which areas of the system are experiencing the most traffic and take steps to optimize them.

Similarly, in a key-value storage system, we might measure transactions and retrievals per second to determine the level of traffic on the system. By monitoring this metric, we can identify any performance issues that are impacting the system and take steps to optimize it for better overall performance.

Overall, measuring traffic is essential for understanding the demand being placed on a system and ensuring that it can handle that demand without performance issues. By monitoring traffic and taking steps to optimize the system as needed, we can provide a better user experience and ensure that the system can scale as demand grows.

## Errors

Let’s say you have a web application that provides online banking services. When customers use the application to transfer funds, check their account balance, or pay bills, the application processes their requests and provides a response. However, not all requests are successful — some may fail due to network errors, database connectivity issues, or other factors.

To measure errors, we need to track the rate of requests that fail. These failures can be explicit, such as an HTTP 500 error indicating a server-side issue. They can also be implicit, such as an HTTP 200 success response that returns the wrong content. Finally, errors can also be based on policy, such as if the system has committed to responding within one second, any request that takes longer than one second is considered an error.

To monitor errors effectively, it’s essential to have clear protocols for tracking and reporting them. In some cases, protocol response codes may not be sufficient to express all failure conditions, and additional internal protocols may be necessary to track partial failure modes.

Monitoring errors can also be different depending on the approach taken. For example, catching HTTP 500 errors at the load balancer can do a decent job of catching all completely failed requests. Still, only end-to-end system tests can detect when the application is serving the wrong content. Therefore, it’s essential to use a combination of different monitoring techniques to catch errors effectively.

Overall, tracking errors is crucial for understanding how well a system is performing and detecting any issues that may be impacting its reliability. By monitoring error rates and taking steps to address any issues, we can ensure that the system is performing as expected and delivering a high-quality user experience.

## Saturation

Let’s say you have an online store that sells products to customers. As your business grows, you start receiving more and more traffic on your website. This increased traffic can lead to performance issues if your website is not able to handle the load.

To monitor saturation, you might want to track the CPU and memory usage of your website’s server(s). This will give you an idea of how much of the available resources are being used at any given time. For example, if your server has 4 CPU cores and 16 GB of RAM, you might set a utilization target of 75% for both CPU and memory usage. This means that if your website’s CPU or memory usage reaches 75%, you might consider scaling up your infrastructure to handle the increased load.

To track saturation, you might set up a dashboard that displays the CPU and memory utilization metrics over time. This will allow you to identify any spikes in usage and take action to prevent performance issues. For example, you might notice that CPU usage consistently spikes to 80% during peak hours. In response, you might consider adding more servers or upgrading your existing servers to handle the increased load.

An increase in latency can also be an indicator of saturation. Let’s say that during peak hours, your website’s response time starts to increase. You might measure the 99th percentile response time over a small time period, such as 5 minutes, and notice that it has increased from 50 ms to 60 ms. This indicates that 1 out of every 100 requests is taking longer to process, likely due to saturation. By monitoring this metric, you can proactively identify and address saturation issues before they become more severe and impact your customers’ experience.

The USE method is a framework used to quickly identify and diagnose performance issues in an unknown system. The acronym USE stands for Utilization, Saturation, and Errors, and these three metrics are used to assess the health and efficiency of a system.

In Kubernetes The USE method is for the Resources

**Utilization** refers to the average time a resource is busy servicing work. It is the percentage of time a resource is busy handling requests or processing data. For example, if a CPU is utilized 80% of the time, it means that it was busy handling requests or executing instructions for 80% of the observed time period.

**Saturation**, on the other hand, refers to the degree to which a resource has extra work that it cannot service, often queued. It measures how full a particular resource is, and whether it is struggling to keep up with the demand placed on it. For example, if a disk is 90% saturated, it means that there is only 10% capacity remaining for new requests or data.

**Errors**, the third metric in the USE method, are the count of error events. They refer to the number of errors generated by the system, whether they are disk read errors, network errors, or other types of errors. Errors can help identify underlying issues in the system, such as hardware failures or software bugs.

By monitoring these three metrics, you can identify potential issues in your system before they become critical. For example, if the utilization of a CPU is consistently above 90%, it may indicate that the CPU is becoming a bottleneck in the system, and it may need to be upgraded. Similarly, if a disk is consistently saturated, it may indicate that there is a problem with the system’s storage capacity, and additional storage may need to be added.

The USE method is intended to be a simple, straightforward, complete, and fast way to diagnose performance issues. By using this framework, you can quickly identify the most likely sources of problems in a system, allowing you to focus your attention on the areas that need the most attention.

The RED method is another approach to monitoring and troubleshooting systems that was developed by Tom Wilkie. RED stands for Rate, Errors, and Duration, and it provides a framework for thinking about the different aspects of a service that you need to monitor to ensure it is healthy and performing well.

The RED method is used mostly in Kubernetes Services.

Here’s what each component of the RED method means:

-   **Rate**: This refers to the number of requests per second that a service is handling. It is important to monitor the rate to ensure that the service can handle the expected load and that it is not being overwhelmed by too many requests.
-   **Errors**: This refers to the number of errors that occur when handling requests. Monitoring errors is important because they can indicate problems with the service that need to be addressed. A high error rate could indicate issues with the service code, infrastructure, or dependencies.
-   **Duration**: This refers to the time it takes for a service to handle a request, from the moment it is received to the moment a response is sent back to the client. Monitoring duration is important because it can indicate performance issues with the service, such as slow response times or bottlenecks.

By monitoring these three components, you can quickly identify issues with services and take steps to address them before they impact your users.

Node exporter is installed as a daemon set whereby each instance is running in a single node

## USE for Node CPU

## Utilization

The series that comes out of the node is called **node\_cpu**

sum(rate{node\_cpu{mode!="idle",mode="iowait"}\[5m\]}) BY (instance)

The query calculates the rate of increase in CPU usage in “iowait” mode by subtracting the current value from the previous value and dividing it by the time elapsed. The “sum” function is then used to aggregate the results by instance. This Query is a counter and measures the CPU utilization over time.

## Saturation

The series used to measure this is called **node\_load1.** The easiest way to talk about saturation is to talk about the load average. Load average, as the name suggests, depicts the average load on a CPU for a set time interval. These values are the number of processes waiting for the CPU or using it in the given period.

sum(node\_load1) by node / count(node\_cpu{mode="system"}) by (node) \* 100

The PromQL expression calculates the average system load per CPU core for each node in a Kubernetes cluster.

The query first calculates the sum of the load average values for the past 1 minute (node\_load1) for each node in the cluster using the “sum” function and grouping the results by node using the “by node” clause.

It then divides this value by the count of the number of CPU cores running in “system” mode for each node in the cluster over the past 1 minute. This is done using the “count” function and filtering the results by the “mode” label being set to “system”. The result of this division is multiplied by 100 to get the percentage of system load per CPU core.

Overall, this query can be used to monitor the system load on each node in a Kubernetes cluster and identify any nodes that are experiencing high load per CPU core, which could indicate a need for additional resources or optimization.

## Errors

The errors are not exposed by the node\_exporter

## USE for Node Memory

## Utilization

The series used to measure the node memory are:

node\_memory\_MemAvailable  
node\_memory\_MemTotal

kube\_node\_status\_capacity\_memory\_bytes  
kube\_node\_status\_allocatable\_memory\_bytes

**Node\_memory** this refers to the set of metrics that track the memory usage of a node in the Kubernetes cluster. The `node_memory` metrics provide insight into the memory utilization patterns of the node, including how much memory is actively being used by running processes, how much memory is being used for file system caching, and how much memory is available for use by applications.

## Example of a PromQL Querry for Node\_Memory

i) Percentage of available memory on each node

sum(node\_memory\_MemAvailable) by (node) / sum(node\_memory\_MemTotal) by (node)

The expression first calculates the total amount of available memory on each node by summing the `node_memory_MemAvailable` metric over each unique `node` label value, using the `sum()` function and the `by(node)` clause. This gives us the total amount of memory available for use by applications on each node.

Next, the expression calculates the total amount of physical memory on each node by summing the `node_memory_MemTotal` metric over each unique `node` label value. This gives us the total amount of memory capacity on each node.

Finally, the expression divides the total amount of available memory by the total amount of memory capacity for each node, giving us the percentage of available memory on each node. This is indicated by the division operator `/`.

By running this query in Prometheus, you can get a visual representation of the percentage of available memory on each node in your cluster. This can be useful for monitoring the overall memory usage patterns on your cluster, identifying nodes that may be experiencing memory pressure, and making informed decisions about capacity planning and scaling.

ii.) Percentage of allocatable memory on each node

sum(kube\_node\_status\_allocatable\_memory\_bytes) by (exported\_node) / sum(kube\_node\_status\_capacity\_memory\_bytes) by (exported\_node)

The expression first calculates the total amount of allocatable memory on each node by summing the `kube_node_status_allocatable_memory_bytes` metric over each unique `exported_node` label value, using the `sum()` function and the `by(exported_node)` clause. This gives us the total amount of memory that can be allocated to pods on each node.

Next, the expression calculates the total amount of physical memory capacity on each node by summing the `kube_node_status_capacity_memory_bytes` metric over each unique `exported_node` label value. This gives us the total amount of memory capacity on each node.

Finally, the expression divides the total amount of allocatable memory by the total amount of memory capacity for each node, giving us the percentage of allocatable memory on each node. This is indicated by the division operator `/`.

By running this query in Prometheus, you can get a visual representation of the percentage of allocatable memory on each node in your cluster. This can be useful for monitoring the overall capacity of your cluster, identifying nodes that may be reaching their resource limits, and making informed decisions about resource allocation and scaling. The continuation of the blog can be found at this [*link*](https://medium.com/@onai.rotich/understand-container-metrics-and-why-they-matter-9e88434ca62a)
