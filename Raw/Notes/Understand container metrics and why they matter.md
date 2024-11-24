---
title: Understand container metrics and why they matter
source: https://medium.com/@onai.rotich/understand-container-metrics-and-why-they-matter-9e88434ca62a
clipped: 2023-09-04
published: 
category: observability
tags:
  - k8s
  - observability
read: false
---

![[Raw/Media/Resources/019f86563ca59ad91a8c9af724596ab7_MD5.png]]

> This is a continuation of the previous [blog](https://medium.com/@onai.rotich/key-metrics-to-scrape-for-effective-monitoring-and-optimization-of-a-kubernetes-cluster-c2dd80568a2e) about the Key metrix to scrape in your cluster . Before going through this blog go through first the previous one to understand more

## cAdvisor

cAdvisor (short for Container Advisor) is an open-source tool for monitoring resource usage and performance metrics of containers and their host systems. It was developed by Google and is now part of the Cloud Native Computing Foundation (CNCF) project.

cAdvisor runs as a daemon process on each node of a container cluster and collects real-time metrics of all containers running on the node such as

i.) CPU usage (user and system) and time throttled

ii.) Memory usage and limits

iii.) Network I/O, and disk I/O and Dropped

iv.) File system read/write

It can also provide a web interface for viewing these metrics, as well as an API for integrating with external monitoring systems such as Prometheus.

cAdvisor is embedded into the kublet and so we scrape the kublet to get container metrix these are so-called Kubernetes “core” metrix

## USE for Container CPU

**Utilization**

The series used to measure the container metrics are

-   `cpu.usage.total`: CPU usage of the container as a percentage of the total CPU available on the host system.
-   `memory.usage`: Memory usage of the container in bytes.
-   `network.usage`: Network usage statistics of the container, including the number of bytes received and transmitted.
-   `diskio.io_service_bytes`: Disk I/O service statistics of the container, including the number of bytes read and written.
-   `filesystem.usage`: Disk usage of the container's file system in bytes.

**Example** of CPU resources consuming by each container

sum(rate{container\_cpu\_usage\_seconds\_total\[5m\]} by (container\_name)

This query calculates the per-second rate of CPU usage for each container over the last 5 minutes (`[5m]`), and then sums up the values for each container (`sum(...) by (container_name)`). The metric being queried is `container_cpu_usage_seconds_total`, which measures the total amount of CPU time used by a container in seconds.

By calculating the rate of CPU usage and aggregating the results by container name, this query provides a useful overview of how much CPU resources each container is consuming, and can help identify containers that are using an unusually high amount of CPU.

**Saturation**

The main series type used are:

-   `container_memory_usage_bytes`: Measures the amount of memory being used by the container. When this value approaches the limit of the container's memory allocation, it can cause saturation and affect the container's performance.
-   `container_cpu_usage_seconds_total`: Measures the amount of CPU time used by the container. If the CPU is fully utilized and unable to handle additional requests, it can lead to saturation.
-   `container_network_transmit_bytes_total` and `container_network_receive_bytes_total`: Measure the amount of network traffic transmitted and received by the container, respectively. When the network becomes saturated, the container may experience latency or dropped requests.
-   `container_fs_usage_bytes`: Measures the amount of file system storage used by the container. If the file system is nearly full, it can cause performance issues and saturation.
-   `container_fs_reads_total` and `container_fs_writes_total`: Measure the number of read and write operations performed on the container's file system. When the file system is heavily used, it can cause saturation and slow down container performance.

**Example**

sum(rate{container\_cpu\_cfs\_throttled\_seconds\_total\[5m\]} by (container\_name)

Breaking down the query, it is calculating the rate at which the CPU of containers has been throttled over the last 5 minutes. Specifically, it is using the `container_cpu_cfs_throttled_seconds_total` metric to calculate this rate. The `sum` function is then used to aggregate the results by `container_name`, which means that the rate will be calculated for each container separately.

To clarify, the `container_cpu_cfs_throttled_seconds_total` metric measures the total time in seconds that a container's CPU usage has been throttled due to being over its CFS (Completely Fair Scheduler) quota. The `rate` function is used to calculate the per-second rate of change in this metric over the last 5 minutes.

Overall, this query can be useful for monitoring the CPU usage of containers and identifying if any of them are being throttled due to exceeding their allocated quota.

## USE for Container Memory

## Utilization

The series used to measure the container memory is `container_memory_usage_bytes` or `container_memory_working_set_bytes`

1.  `container_memory_usage_bytes` - This metric tracks the total amount of memory used by a container, including both anonymous and page cache memory.
2.  `container_memory_rss` - This metric tracks the amount of memory used by a container's working set, which includes all pages the container has accessed recently.
3.  `container_memory_cache` - This metric tracks the amount of memory used by a container's page cache.
4.  `container_memory_swap` - This metric tracks the amount of swap space used by a container.

Example

sum(container\_memory\_working\_set\_bytes{name!~POD}) by (name)

So, the overall query is calculating the sum of the working set memory usage for each container, excluding Pods, and grouping the results by the `name` label. This can be useful for monitoring the memory usage of individual containers and identifying which containers may be using more memory than expected.

It’s worth noting that while the working set metric is useful for monitoring memory usage, it does not include other types of memory usage such as cache or swap usage. Therefore, it may be necessary to use additional metrics or queries to get a complete picture of container memory usage.

## **Saturation**

For example, let's say you have set limits of resources the container has to use and now you want to calculate how close you are to the limits you set out. This is worked out using the Rate function where we rate the container\_memory\_working\_set\_byte / kube\_pod\_container\_resource\_limits\_memory\_bytes

sum(container\_memory\_working\_set\_byte) by {container\_name} / sum(label\_join{kube\_pod\_container\_resource\_limits\_memory\_bytes, "container\_name", "","container"}) by (container\_name)

-   `sum(container_memory_working_set_byte) by {container_name}` calculates the total working set memory usage for each container, and groups the result by the `container_name` label.
-   `sum(label_join{kube_pod_container_resource_limits_memory_bytes, "container_name", "","container"}) by (container_name)` calculates the total memory limit for each container, and groups the result by the `container_name` label. This uses the `label_join` function to join the `kube_pod_container_resource_limits_memory_bytes` metric with the `container_name` label, so that the result is grouped by container.
-   Finally, the two results are divided by each other to calculate the memory utilization percentage for each container.

This query can be useful for monitoring the memory usage of individual containers and identifying which containers may be using a high percentage of their allocated memory.

## Errors

The series mainly used in errors is container\_memory\_failures\_total `container_memory_failures_total` is a metric in Prometheus that tracks the number of times a container has experienced an out-of-memory (OOM) event. An OOM event occurs when a container attempts to use more memory than is available to it, causing the system to terminate the container.

The `container_memory_failures_total` metric can be used to monitor the health of containers and to identify any memory-related issues that may be causing problems in a system. It can also be used to trigger alerts or notifications when the number of OOM events exceeds a certain threshold.

sum(rate{container\_memory\_failures\_total{type\="pagmajfault"}\[5m\]}) by (container\_name)

The `container_memory_failures_total` metric tracks the number of memory-related failures for containers, and the `type="pagmajfault"` selector filters the metric to only include major page faults. The `rate()` function calculates the per-second rate of change of the metric over the 5-minute time window. Finally, the `by (container_name)` clause groups the result by the container name, so that you get a separate time series for each container.

Overall, this query can help you monitor the rate of major page faults occurring in your containers, which can be an important indicator of system health and performance. By grouping the results by container, you can identify any specific containers that are experiencing a high rate of major page faults and investigate further to identify and address any issues.

## pagmajfault

`pagmajfault` metric can be collected for containers using the `container_memory_major_page_faults_total` metric. This metric tracks the number of major page faults that have occurred in a container since it was started.

## majfault

`majfault container_page_faults_major_total` metric for each container in your Kubernetes cluster.

For example, the following Prometheus query can be used to calculate the rate of major page faults per second for each container over a 5-minute window:

sum(rate(container\_page\_faults\_major\_total\[5m\])) by (container\_name)

This query will give you a time series of the rate of major page faults per second for each container in your Kubernetes cluster.

Metrics about the performance of K8s API Server these are the series that we will find in the API server.

## Kubernetes API Server:

-   `apiserver_request_duration_seconds`: measures the latency of API requests to the Kubernetes API server.
-   `apiserver_request_total`: tracks the total number of API requests made to the Kubernetes API server.
-   `apiserver_current_inflight_requests`: measures the number of in-flight API requests to the Kubernetes API server.
-   `apiserver_audit_event_total`: tracks the total number of audit events generated by the Kubernetes API server.
-   `apiserver_longrunning_gauge`: measures the number of long-running API requests to the Kubernetes API server.

## **Controller Worker Nodes:**

-   `kubelet_running_pod_count`: measures the number of running pods on each worker node.
-   `kubelet_container_log_filesystem_used_bytes`: tracks the amount of storage used by container logs on each worker node.
-   `kubelet_node_status_capacity_cpu_cores`: measures the total CPU capacity of each worker node.
-   `kubelet_node_status_capacity_memory_bytes`: measures the total memory capacity of each worker node.
-   `kubelet_node_status_allocatable_cpu_cores`: measures the amount of CPU resources available for pods on each worker node.
-   `kubelet_node_status_allocatable_memory_bytes`: measures the amount of memory resources available for pods on each worker node.

## Etcd Helper Cache:

-   `etcdhelper_cache_work_duration_seconds`: measures the duration of etcd helper cache work queries.
-   `etcdhelper_cache_work_errors_total`: tracks the total number of errors encountered during etcd helper cache work queries.
-   `etcdhelper_cache_hits_total`: tracks the total number of successful cache hits in the etcd helper cache.
-   `etcdhelper_cache_misses_total`: tracks the total number of cache misses in the etcd helper cache.
-   `etcdhelper_cache_size_bytes`: measures the size of the etcd helper cache in bytes.

## General Process Status:

-   `process_open_fds`: measures the number of open file descriptors for a given process.
-   `process_resident_memory_bytes`: measures the amount of resident memory used by a given process.
-   `process_cpu_seconds_total`: tracks the total CPU usage of a given process.

## Golang Status:

-   `go_gc_duration_seconds`: measures the duration of Go garbage collection cycles.
-   `go_memstats_alloc_bytes`: measures the amount of memory allocated by the Go runtime.
-   `go_threads`: measures the number of threads used by the Go runtime.

From the Above let's check some examples in the RED Method.

`apiserver_request_count`

sum(rate{apiserver\_request\_count\[5m\]}) by (verb)

The query calculates the per-second request rate for each API server request verb (e.g. GET, POST, PUT, DELETE) over a 5-minute period.

Here’s how the query works:

-   `rate{apiserver_request_count[5m]}` calculates the per-second rate of API server requests over the last 5 minutes.
-   `sum(...)` calculates the total request rate for each API server request verb by summing the per-second request rates for all requests with the same verb.
-   `by (verb)` groups the request rates by verb, so that the query returns a separate time series for each verb.

This query can be useful for understanding the distribution of API server requests across different request verbs and identifying any verbs that may be responsible for a disproportionately high number of requests. By monitoring the per-second request rate for each verb, you can also detect changes in the traffic patterns for your Kubernetes cluster over time and take appropriate actions to optimize the performance and scalability of your cluster.

`apiserver_request_latency_bucket`

histogram\_quantile(0.9, rate(apiserver\_request\_latency\_bucket\[5m\]))1e+06

The query you provided `histogram_quantile(0.9, rate(apiserver_request_latency_bucket[5m]))1e+06` calculates the 90th percentile (i.e. the value below which 90% of the data falls) of the distribution of API server request latencies over a 5-minute period.

Here’s how the query works:

-   `rate(apiserver_request_latency_bucket[5m])` calculates the per-second rate of API server requests over the last 5 minutes for each latency bucket defined in the `apiserver_request_latency_bucket` histogram.
-   `histogram_quantile(0.9, ...)` calculates the 90th percentile of the latency distribution by aggregating the per-second request rates for each bucket and interpolating between them.

The `1e+06` at the end of the query is a unit conversion factor that converts the latency value from seconds to microseconds.

This query can be useful for understanding the performance of the Kubernetes API server and identifying any issues that may be causing slow response times for API requests. By monitoring the 90th percentile latency over time, you can detect changes in the API server’s performance and take appropriate actions to optimize its performance and scalability.

-   Counts and metadata about many Kubernetes types: `kube-state-metrics` exposes metrics for various Kubernetes types such as pods, deployments, statefulsets, and services. These metrics can provide information such as the number of running or pending pods, the number of replicas for a deployment or statefulset, and the metadata labels and annotations for these resources.
-   Counts of many Kubernetes resources: In addition to types, `kube-state-metrics` also provides metrics for resource counts such as the number of nodes, namespaces, pods, services, and config maps in the cluster.
-   Resource limits: `kube-state-metrics` exposes metrics that provide information about the resource limits and requests for Kubernetes pods, such as CPU and memory limits and requests.
-   Container states: `kube-state-metrics` exposes metrics that provide information about the state of Kubernetes containers, such as the number of restarts, the container uptime, and the container termination reason.

By using these derived metrics, you can gain visibility into the state of your Kubernetes cluster and track resource usage and performance over time. This can help you identify and troubleshoot issues related to container and pod health, resource utilization, and application performance.

![[Raw/Media/Resources/f84a17062ec72450d9279ad92ec59b68_MD5.png]]

Etcd is a distributed key-value store that is used by Kubernetes for storing cluster state information. Etcd provides a set of metrics that can be exposed to Prometheus for monitoring and troubleshooting the performance of the etcd cluster.

Some of the key etcd metrics include:

-   etcd\_disk\_wal\_fsync\_duration\_seconds: Measures the duration of disk write-ahead log (WAL) fsyncs in seconds.
-   etcd\_disk\_backend\_commit\_duration\_seconds: Measures the duration of backend commit calls in seconds.
-   etcd\_server\_has\_leader: Indicates whether the etcd cluster has a leader node.
-   etcd\_server\_proposal\_failed\_total: Counts the number of failed proposals to the etcd cluster.
-   etcd\_server\_proposals\_committed\_total: Counts the number of proposals committed to the etcd cluster.
-   etcd\_network\_client\_sent\_bytes\_total: Counts the total number of bytes sent by etcd clients.
-   etcd\_network\_peer\_sent\_bytes\_total: Counts the total number of bytes sent by etcd peers.

By monitoring these metrics, you can gain insight into the performance and health of the etcd cluster. For example, high values for etcd\_disk\_wal\_fsync\_duration\_seconds or etcd\_disk\_backend\_commit\_duration\_seconds may indicate that disk I/O is a bottleneck in the cluster. Similarly, a high number of failed proposals or low etcd\_server\_has\_leader values may indicate that the cluster is experiencing issues with leadership election or network connectivity.

## Inbound gRPC stats:

Inbound gRPC stats are related to the kube-apiserver and include metrics such as the number of inbound RPCs, the total amount of data received over gRPC, and the duration of inbound RPCs. These metrics can be used to monitor the performance and usage of the kube-apiserver and to identify any issues related to latency, network connectivity, or RPC errors.

## Intra-cluster gRPC stats:

Intra-cluster gRPC stats are related to the communication between the kube-scheduler and kube-controller-manager. These metrics include the number of gRPC messages sent and received, the total number of bytes transferred over gRPC, and the duration of gRPC calls. These metrics can be used to monitor the health and performance of the kube-scheduler and kube-controller-manager, and to identify any issues related to communication or network connectivity between these components.

By monitoring these metrics, you can gain insight into the performance and health of the various components in your Kubernetes cluster that use gRPC for communication. This can help you detect and troubleshoot issues related to network connectivity, latency, or errors in the gRPC communication layer.

