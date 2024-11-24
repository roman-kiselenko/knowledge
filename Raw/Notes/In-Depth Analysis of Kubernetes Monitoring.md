---
title: In-Depth Analysis of Kubernetes Monitoring
source: https://zivukushingai.medium.com/in-depth-analysis-of-kubernetes-monitoring-7b3c1f442542
clipped: 2023-09-15
published: 
category: observability
tags:
  - k8s
read: false
---

In Kubernetes, monitoring is an important aspect that can help you understand the health, performance, and availability of your clusters.

![[Raw/Media/Resources/def8584644a7663e1fd79db0ae5bd8b6_MD5.jpg]]

Photo by [Luke Chesser](https://unsplash.com/@lukechesser?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral)

Depending on the type and scope of monitoring, Kubernetes monitoring can be divided into the following types:

-   Cluster level monitoring
-   Node Layer Monitoring
-   Application Layer Monitoring
-   Log Monitoring

## Cluster Layer Monitoring

Cluster layer monitoring refers to monitoring the entire Kubernetes cluster, including nodes, Pods, and services. Cluster layer monitoring usually focuses on cluster resource usage, load balancing, service discovery, and other indicators to help users understand the overall health and performance of the cluster.

Commonly used cluster layer monitoring tools include Prometheus, Grafana, Heapster, etc.

## Monitoring Principle

Kubernetes cluster-level monitoring is usually implemented based on the principles of metric collection and storage. The basic process is as follows:

-   Metric collection
-   Metric storage
-   Data visualization

## Metric collection

Kubernetes cluster components and objects produce various metrics, such as CPU, memory, disk, and network usage. These metrics can be collected using a variety of methods, including the Kubernetes API, container runtimes, and system monitoring tools. The collection method and frequency will vary based on the cluster's needs and resource constraints.

## Metric storage

The collected metrics need to be stored and managed for subsequent querying and analysis. Commonly used metric storage tools include Prometheus, InfluxDB, and Elasticsearch. These tools can store the collected metrics in a database and provide query and analysis interfaces to help you understand the status and health of the cluster.

## Data visualization

The stored metric data can be visualized using data visualization tools to help users understand the status and health of the cluster more intuitively. Commonly used data visualization tools include Grafana, Kubernetes Dashboard, and Kibana. These tools can transform metric data into charts, graphs, and dashboards that can be used to identify and troubleshoot problems.

## Node Layer Monitoring

Node-level monitoring refers to monitoring the health and performance of Kubernetes nodes. This includes monitoring the node’s CPU, memory, disk, network, and other metrics. Node-level monitoring usually focuses on indicators such as node load status, resource usage, and container status. This information can help users identify and troubleshoot problems with the node.

Commonly used node-level monitoring tools include cAdvisor and Node Exporter.

## Application Layer Monitoring

Application layer monitoring refers to monitoring the health and performance of applications running in Kubernetes. This includes monitoring the application’s CPU, memory, network, disk, and other metrics. Application layer monitoring usually focuses on indicators such as application performance, errors, logs, and other metrics. This information can help users identify and troubleshoot problems with the application.

Commonly used application layer monitoring tools include Prometheus, ELK Stack, and Zipkin.

## Log Monitoring

Monitoring logs is the process of tracking logs produced within Kubernetes clusters, including container, system, and application logs. The objective of log monitoring is to analyze the format, content, and volume of the logs to provide users with insights into the performance and issues of the cluster.

Commonly used log monitoring tools include ELK Stack and Fluentd.

## Monitoring Techniques and Tools

The following are some commonly used cluster-level monitoring technologies and tools:

1.  Prometheus + Grafana
2.  Heapster
3.  Kubernetes Dashboard
4.  Kube-state-metrics

## Prometheus + Grafana

Prometheus and Grafana are common monitoring tools in Kubernetes that enable tracking of resource usage and cluster health status.

Prometheus can collect and store various cluster indicator data, such as CPU, memory, disk, and network, and Grafana can convert these data into beautiful charts and dashboards to help users quickly find and solve problems.

In addition, Prometheus also provides some built-in cluster monitoring rules to help users monitor the status and health of the cluster.

## **Install**

To install Prometheus and Grafana using docker-compose:

1.  Create a directory to store the docker-compose.yml file and related configuration files, such as prometheus-grafana.yml.
2.  Create the docker-compose.yml file in this directory and add the following content:

```yaml
version: '3.7'  
services:  
  prometheus:  
    image: prom/prometheus  
    container\_name: prometheus  
    ports:  
      \- "9090:9090"  
    volumes:  
      \- ./prometheus.yml:/etc/prometheus/prometheus.yml  
    command:  
      \- \--config.file=/etc/prometheus/prometheus.yml  
      \- \--storage.tsdb.path=/prometheus  
  grafana:  
    image: grafana/grafana  
    container\_name: grafana  
    ports:  
      \- "3000:3000"  
    volumes:  
      \- grafana-storage:/var/lib/grafana  
volumes:  
  grafana-storage:
```

This docker-compose.yml file defines two services, Prometheus and Grafana. Prometheus will expose port 9090, and Grafana will expose port 3000.

Both services will be configured using the configuration file in the directory where the docker-compose.yml file is located.

Create the prometheus.yml file in this directory and add the following content:

```yaml
global:  
  scrape\_interval: 15s  
scrape\_configs:  
  \- job\_name: 'prometheus'  
    scrape\_interval: 5s  
    static\_configs:  
      \- targets: \['localhost:9090'\]  
  \- job\_name: 'node'  
    scrape\_interval: 5s  
    static\_configs:  
      \- targets: \['node-exporter:9100'\]  
  \- job\_name: 'cadvisor'  
    scrape\_interval: 5s  
    static\_configs:  
      \- targets: \['cadvisor:8080'\]  
  \- job\_name: 'kubelet'  
    scrape\_interval: 5s  
    static\_configs:  
      \- targets: \['kubelet:10255'\]
```

The prometheus.yml file defines the basic scrape configs for collecting metrics from Kubernetes nodes, cAdvisor, kubelet, and Prometheus itself.

To start the Prometheus and Grafana services, you can run the following commands in the directory where the docker-compose.yml file is located:

`docker-compose up -d`

This command will start the Prometheus and Grafana containers in detached mode.

1.  Open the browser and visit `http://localhost:3000` to enter the Grafana login interface. Log in using the default username and password (admin/admin).
2.  Add data source in Grafana. Select Configuration->Data Sources in the left menu bar and click the Add data source button. Select Prometheus as the data source type, then fill in the Prometheus address `http://prometheus:9090`, and click the Save & Test button to test whether the connection is successful.
3.  Import Dashboard in Grafana. Grafana provides many ready-made Dashboards that users can select and import according to their needs. Select + -> Import in the left menu bar, then select a Dashboard template file.

Prometheus and Grafana have been successfully installed and configured using docker-compose. You can now use Grafana to view and analyze the Kubernetes cluster metrics data collected by Prometheus.

## Heapster

Heapster is a Kubernetes cluster monitoring and performance analysis tool. It can collect resource usage data from all containers in the cluster and store it in a specified backend storage, such as InfluxDB and Elasticsearch. Heapster also provides command-line tools and APIs for users to query and analyze the data, including node, Pod, and container data.

## Kubernetes Dashboard

Kubernetes Dashboard is a web-based user interface (UI) for managing and monitoring Kubernetes clusters. It provides a single point of access to view cluster-level metrics, such as resource usage, Pod and container status, and events. The dashboard also provides tools for managing Pods and deployments, as well as viewing logs and debugging issues.

## Kube-state-metrics

Kube-state-metrics is a Kubernetes metrics exporter that collects and exposes metrics about the state of Kubernetes objects. It can export metrics about objects such as nodes, Pods, services, replica sets, Deployments, and DaemonSets. These metrics can be used to understand the status and health of the cluster.

The best tool or technology for you will depend on your specific needs and requirements. You may need to use multiple tools to fully understand your cluster's status.

## etcd Watch Monitoring Techniques

## etcd Watch Monitoring

In Kubernetes, etcd is a distributed key-value store used to store configuration data and status information of the cluster. etcd provides some API interfaces, including the watch interface, which can be used to monitor changes in the directories specified in etcd. etcd’s watch mechanism is very powerful and can help users achieve real-time configuration updates and status synchronization.

The following are some techniques and practices about etcd’s watch monitoring:

## etcd Directory Monitor

You can use the `etcdctl` command or the etcd client library to monitor key value changes in the specified directory in `etcd`. For example, to use `etcdctl` to monitor changes in the `/mydir` directory, use the following command:

etcdctl watch /mydir

This command will start watching the `/mydir` directory for changes. When the key-value pair in the directory `/mydir`changes, etcd will notify the listener of the change.

## etcd Client Library

etcd client libraries provide some advanced watch mechanisms that can be used to implement more flexible monitoring of etcd. For example, you can use etcd’s Go client library to implement monitoring of etcd directories by using the Watch function.

Here is an example:

```go
watcher := clientv3.NewWatcher(client)  
watcher.Watch(context.Background(), "/mydir", clientv3.WithPrefix(), clientv3.WithPrevKV())  
for {  
    select {  
    case resp := <-watcher.WatchChan():  
        for \_, event := range resp.Events {  
            fmt.Printf("Event received! Type: %s Key: %s Value: %s\\n", event.Type, event.Kv.Key, event.Kv.Value)  
        }  
    }  
}
```

In the above example, a Watcher instance is created using the etcd Go client library, and then the Watch function is called to monitor changes in the `/mydir` directory. When the key value in the directory changes, Watcher will notify the change to the WatchChan channel, thereby achieving real-time updates and synchronization.

## etcd API Interface

In addition to the etcd client library, you can also use the etcd API interface to monitor etcd. etcd API interface provides some advanced watch functions such as cancelable watches, and multiplexed watches.

The following is an example implemented using etcd’s API interface:

```go
watcher := clientv3.NewWatcher(client)  
ctx, cancel := context.WithCancel(context.Background())  
watcher.Watch(ctx, "/mydir", clientv3.WithPrefix())  
go func() {  
    for {  
        select {  
        case resp := <-watcher.Chan():  
            for \_, event := range resp.Events {  
                fmt.Printf("Event received! Type: %s Key: %s Value: %s\\n", event.Type, event.Kv.Key, event.Kv.Value)  
            }  
        }  
    }  
}()  
  
time.Sleep(10 \* time.Second)  
cancel()
```

In the above example, a Watcher instance is created using etcd’s API interface, and then the Watch function is called to monitor changes in the `/mydir` directory.

A cancelable context object is created using the `context.WithCancel` function to cancel listening when needed.

When the key-value in the directory changes, Watcher will notify the change to the Chan channel, thereby achieving real-time updates and synchronization.

etcd watch mechanism can help you achieve real-time configuration updates and status synchronization. According to actual needs and scenarios, you can choose appropriate monitoring methods and technologies to monitor etcd.