---
title: OpenTelemetry Operator
source: https://medium.com/@magstherdev/opentelemetry-operator-d3d407354cbf
clipped: 2023-09-04
published: 
category: observability
tags:
  - k8s
  - observability
read: true
---

## Introduction

This post aims to demonstrate how you can implement traces in your application **without any code changes** by using the **OpenTelemetry Operator.**

Before we start with that, I’d recommend you to read some of my previous post about **OpenTelemetry**. You can find them [here](https://faun.pub/opentelemetry-d71d369c83d7), [here](https://medium.com/@magstherdev/opentelemetry-on-kubernetes-c167f024b35f), [here](https://faun.pub/opentelemetry-backends-240acbd8dcac) and [here](https://faun.pub/opentelemetry-and-spring-boot-b2affa80ffec).

Today, we will begin creating a deployment manifest for a **Java** application (Spring [PetClinic](https://github.com/spring-petclinic/spring-framework-petclinic)) and run it on a **Kubernetes** cluster.

After that, we will deploy the **OpenTelemetry Operator**. The operator will provide us with two custom resources that we will use to **auto-instrument** our application.

At the end we will **visualise** the traces with **Grafana Tempo**, but you can easily switch that to a vendor of choice.

The **workflow** will look like this:

![[Raw/Media/Resources/750f7564838d47743464a5193698c761_MD5.png]]

> In the [**OpenTelemetry Backends — Which vendor should you choose?**](https://faun.pub/opentelemetry-backends-240acbd8dcac) post**,** we covered the following vendors: [Honeycomb](https://medium.com/@magstherdev/honeycomb-8014420a6f7b), [Lightstep](https://medium.com/@magstherdev/lightstep-5bd0a8e060d), [New Relic](https://medium.com/@magstherdev/new-relic-20f97b9f9d1d) and [Tempo (Grafana Cloud)](https://medium.com/@magstherdev/grafana-cloud-tempo-3b95373ff9d0)

## OpenTelemetry Collector — Deployment Patterns / Strategy

The OpenTelemetry collector can be deployed in different ways and it is important to think of how you want to deploy it.

> The strategy that you pick is often depending on the teams and organisations.

**OpenTelemetry Agent to a Backend**

In this scenario, the “**Agent mode”;** the OTEL instrumented application sends the telemetry data to a (collector) **agent** that resides together with the application or on the same host as the application, as a **Sidecar** or **DaemonSet**.

One of the advantages in this scenario, is that the agent will **offload** responsibility and handle all the trace data from the instrumented application.

Here, the **collector** is deployed as an agent via a **sidecar** that sends telemetry data directly to the storage backend(s).

![[Raw/Media/Resources/d7487ffe0876b5ef99784533c317106a_MD5.png]]

**Central (Gateway) OpenTelemetry Collector**

Another strategy is to use a standalone **central (Gateway) collector**. Telemetry data is then received from agents and processed, and exported to a storage backend.

This central (Gateway) OpenTelemetry collector is often deployed using the `deployment` mode, which comes with features like auto scaling.

**Some of the advantages of using a central collector are:**

-   Removes dependancy on the teams
-   General configuration for batching, retry, encryption, compression
-   Authentication
-   Enriching with metadata.

In addition to this, it can perform sampling here depending on the amount of traces/traffic you want to send. (**ie. take only 10% of the traces**).

## Application

To showcase how easy it is to send traces using the OpenTelemetry Operator, we will again use the [Petclinic](https://github.com/spring-projects/spring-petclinic) application, *which is a* [*Spring Boot*](https://spring.io/guides/gs/spring-boot) *application built using* [*Maven*](https://spring.io/guides/gs/maven/) *or* [*Gradle*](https://spring.io/guides/gs/gradle/)*.*

The deployment file for this application look like this:

apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: petclinic  
  labels:  
    app: petclinic  
spec:  
  replicas: 1  
  selector:  
    matchLabels:  
      app: petclinic  
  template:  
    metadata:  
      labels:  
        app: petclinic  
    spec:  
      containers:  
      \- name: petclinic  
        image: <path\_to\_petclinic\_image>   
        ports:  
        \- containerPort: 8080

## OpenTelemetry Operator

As mentioned in the introduction, we will demonstrate how you can implement traces in your application without any code changes by using the **OpenTelemetry Operator.**

This [operator](https://github.com/open-telemetry/opentelemetry-operator) is an implementation of a [Kubernetes Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) *and manages the OpenTelemetry collector and the auto-instrumentation of the workloads using OpenTelemetry instrumentation libraries.*

One of the benefits with using Operators it that it extends the functionally of Kubernetes. The Cloud Native Computing Foundation (CNCF) wrote a good blog [post](https://www.cncf.io/blog/2022/06/15/kubernetes-operators-what-are-they-some-examples/) about this if you want to read more about Kubernetes Operators.

## Deploying the OpenTelemetry Operator

Before we deploy the OpenTelemetry Operator, we will first start deploy another operator, namely the [cert-manager](https://cert-manager.io/docs/installation/).

This can be installed as follows:

kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml

> You can learn more about CertManager in the [How to use CertManager and Let’s encrypt in Kubernetes](https://faun.pub/lets-encrypt-and-certmanager-aa88775730b8) post.

Now that the cert-manager is deployed, we can begin installing the OpenTelemetry collector. This can be done by applying the operator manifest directly like this:

kubectl apply -f https://github.com/open\-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml

You can also install the Opentelemetry Operator via [Helm Chart](https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-operator) from the opentelemetry-helm-charts repository. More information is available in [here](https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-operator).

Once the operator is deployed, it will **provide two custom resources:**

1.  **Collector CRD** (responsible for the collection, processing and transporting of the data)
2.  **Instrumentation CRD** (responsible for auto instrumentation of the application)

`kubectl get crd`

opentelemetrycollectors.opentelemetry.io  
instrumentations.opentelemetry.io

![[Raw/Media/Resources/0de69b8fd316de2f08e2de4ceae91612_MD5.png]]

## Deployment

In our **strategy**, we have chosen to use a **central OpenTelemetry collector** and have other OpenTelemetry agents send their data to this one. The data that is being received from the agents will be processed on this collector and sent over to the storage backend(s) via **exporters**.

![[Raw/Media/Resources/750f7564838d47743464a5193698c761_MD5.png]]

Create a new file and call it `central_collector.yaml`

The example file can be found on the Getting Started [page](https://github.com/open-telemetry/opentelemetry-operator#getting-started).

```yaml
apiVersion: opentelemetry.io/v1alpha1  
kind: OpenTelemetryCollector  
metadata:  
  name: simplest  
spec:  
  config: |  
    receivers:  
      otlp:  
        protocols:  
          grpc:  
          http:  
    processors:  
      memory\_limiter:  
        check\_interval: 1s  
        limit\_percentage: 75  
        spike\_limit\_percentage: 15  
      batch:  
        send\_batch\_size: 10000  
        timeout: 10s  
  
    exporters:  
      logging:

    service:  
      pipelines:  
        traces:  
          receivers: \[otlp\]  
          processors: \[\]  
          exporters: \[logging\]  
EOF
```

Now deploy it using:

`kubectl apply -f central_collector.yaml`

In this example, the OpenTelemetry Collector exposes the `grpc` and `http` ports to consume spans from the instrumented applications and exporting those spans via `logging`, which writes the spans to the console (`stdout`) of the OpenTelemetry Collector instance that receives the span.

## Adding additional exporters

In order to visualise and analyse the telemetry you will need to use an exporter. An **exporter** is a component of OpenTelemetry and is *how data gets sent to different systems/back-ends.*

In the **exporters** section, you can add more destinations. For example, if you would also like to send trace data to **Grafana Tempo**, just add these lines to the `central_collector.yaml` file.

exporters:  
      logging:  
      otlp:  
        endpoint: "<tempo\_endpoint>"  
        headers:  
          authorization: Basic <api\_token>

Also make sure to add this exporter to the `pipeline`:

pipelines:  
        traces:  
          receivers: \[otlp\]  
          processors: \[\]  
          exporters: \[logging, otlp\]

## Sidecar

Now we will deploy an **OpenTelemetry agent** using the **Sidecar** mode. This agent will send traces from the application to our central(gateway) OpenTelemetry collector.

Create a new file and call it `sidecar.yaml`

apiVersion: opentelemetry.io/v1alpha1  
kind: OpenTelemetryCollector  
metadata:  
  name: sidecar  
spec:  
  mode: sidecar  
  config: |  
    receivers:  
      otlp:  
        protocols:  
          grpc:  
          http:  
    processors:  
      batch:  
    exporters:  
      logging:  
      otlp:  
        endpoint: "<path\_to\_central\_collector>:4317"  
    service:  
      telemetry:  
        logs:  
          level: "debug"  
      pipelines:  
        traces:  
          receivers: \[otlp\]  
          processors: \[\]  
          exporters: \[logging, otlp\]

Deploy the sidecar with `kubectl apply -f sidecar.yaml`

## Auto Instrumentation

The [operator](https://github.com/open-telemetry/opentelemetry-operator#opentelemetry-auto-instrumentation-injection) can inject and configure OpenTelemetry auto-instrumentation libraries.

Currently **DotNet**, **Java**, **NodeJS** and **Python** are supported.

To use auto-instrumentation, configure an `Instrumentation` resource with the configuration for the SDK and instrumentation.

For the **Java** application, that will look like this.

`vim instrumentation.yaml`

apiVersion: opentelemetry.io/v1alpha1  
kind: Instrumentation  
metadata:  
  name: java-instrumentation  
spec:  
  propagators:  
    \- tracecontext  
    \- baggage  
    \- b3  
  sampler:  
    type: always\_on  
  java:

If you would use a **Python** application, it will look like this:

apiVersion: opentelemetry.io/v1alpha1  
kind: Instrumentation  
metadata:  
  name: python-instrumentation  
spec:  
  propagators:  
    \- tracecontext  
    \- baggage  
    \- b3  
  sampler:  
    type: always\_on  
  python:

**Samplers and Propagators**

From the example above, we added **propagators** (**tracecontext**, **bagage**, **b3**). These values are added to the `OTEL_PROPAGATORS` environment variable. Valid values for `propagators` are defined by the [OpenTelemetry Specification for OTEL\_PROPAGATORS](https://opentelemetry.io/docs/concepts/sdk-configuration/general-sdk-configuration/#otel_propagators).

> A `*Propagator*` type defines the restrictions imposed by a specific transport and is bound to a data type, in order to propagate in-band context data across process boundaries.

We also defined a **sample** and set this to `always_on` which means it will always samples spans, regardless of the parent span’s sampling decision.

The value for `sampler.type` is added to the `OTEL_TRACES_SAMPLER` envrionment variable. Valid values for `sampler.type` are defined by the [OpenTelemetry Specification for OTEL\_TRACES\_SAMPLER](https://opentelemetry.io/docs/concepts/sdk-configuration/general-sdk-configuration/#otel_traces_sampler).

> Sampling is a mechanism to control the noise and overhead introduced by OpenTelemetry by reducing the number of samples of traces collected and sent to the backend.

## Enabling the instrumentation

To enable the instrumentation, we need to update the deployment file and add **annotations** to it. This way we tell the OpenTelemetry Operator to inject the **sidecar** and the **java-instrumentation** to our application.

Add these lines to the applications deployment file.

annotations:  
  instrumentation.opentelemetry.io/inject-java: 'true'  
  sidecar.opentelemetry.io/inject: 'true'

`petclinic-deployment.yaml`

apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: petclinic  
  labels:  
    app: petclinic  
spec:  
  replicas: 1  
  selector:  
    matchLabels:  
      app: petclinic  
  template:  
    metadata:  
      annotations:  
        instrumentation.opentelemetry.io/inject-java: 'true'  
        sidecar.opentelemetry.io/inject: 'true'  
      labels:  
        app: petclinic  
    spec:  
      containers:  
      \- name: petclinic  
        image: <path\_to\_petclinic\_image>  
        ports:  
        \- containerPort: 8080

## Showtime

Let’s try this all out, by applying all of the files that we created above.

I**nstrumentation**: `kubectl apply -f instrumentation.yaml`

**Sidecar**: `kubectl apply -f sidecar.yaml`

**Application**: `kubectl apply -f petclinic-deployment.yaml`

You can use the **port-forward** command to open the Petclinic UI.

`kubectl port-forward svc/petclinic-svc 8080:80`

To generate traces for the Spring Boot application, access the application on [http://localhost:8080/](http://localhost:8080/)

![[Raw/Media/Resources/38627f05f2bb8020ed0dc66ab5c92b93_MD5.png]]

## Analyse and Query the data

Now to the fun part. How can we view all our traces in a UI?

*Wait, we can’t use OpenTelemetry to view the data?*

No, the OpenTelemetry collector **does not provides their own backend**, so it is up for any vendor or open source product to grab!

In the `central_collectory.yaml` we added an extra exporter. In this configuration we added an exporter for **Grafana Tempo.**

If you now to go the **Grafana UI** and click on the **Explore** button. In the **Datasource** dropdown list, you need to pick the `traces` **datasource**. Select your **service** and click on **Run Query.**

This will show you all trace data for the application.

![[Raw/Media/Resources/49a1cf1906304877ea7bd601f6a94927_MD5.png]]

You can read more about Grafana Tempo [here](https://grafana.com/docs/tempo/latest/).

## Troubleshooting

If you are running into any issues with your deployment, I’d start looking into the logs of the application, and of the containers. There should be two containers , one for the application `petclinic` and one for the collector `otc-container`.

First use the `kubectl describe` command. This command provides detailed information about each of the pods.

`kubectl describe pod petclinic-<pod_name>`

The logs of the petclinic application: `kubectl logs petclinic-<pod_name> -c petclinic`

As you can see, the application (pod) starts and the new pod will be instrumented with OpenTelemetry Java auto-instrumentation.
```
Picked up JAVA\_TOOL\_OPTIONS:  \-javaagent:/otel-auto-instrumentation/javaagent.jar  
OpenJDK 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended  
\[otel.javaagent 2023-03-22 12:33:19:325 +0000\] \[main\] INFO io.opentelemetry.javaagent.tooling.VersionLogger - opentelemetry-javaagent - version: 1.23.0

              |\\      \_,,,--,,\_  
             /,\`.-'\`'   .\_  \\-;;,\_  
  \_\_\_\_\_\_\_ \_\_|,4-  ) )\_   .;.(\_\_\`'-'\_\_     \_\_\_ \_\_    \_ \_\_\_ \_\_\_\_\_\_\_  
 |       | '---''(\_/.\_)-'(\_\\\_)   |   |   |   |  |  | |   |       |  
 |    \_  |    \_\_\_|\_     \_|       |   |   |   |   |\_| |   |       | \_\_ \_ \_  
 |   |\_| |   |\_\_\_  |   | |       |   |   |   |       |   |       | \\ \\ \\ \\  
 |    \_\_\_|    \_\_\_| |   | |      \_|   |\_\_\_|   |  \_    |   |      \_|  \\ \\ \\ \\  
 |   |   |   |\_\_\_  |   | |     |\_|       |   | | |   |   |     |\_    ) ) ) )  
 |\_\_\_|   |\_\_\_\_\_\_\_| |\_\_\_| |\_\_\_\_\_\_\_|\_\_\_\_\_\_\_|\_\_\_|\_|  |\_\_|\_\_\_|\_\_\_\_\_\_\_|  / / / /  
 ==================================================================/\_/\_/\_/  
  
:: Built with Spring Boot :: 3.0.4

2023-03-22T12:33:22.250Z  INFO 1 \--- \[           main\] o.s.s.petclinic.PetClinicApplication     : Starting PetClinicApplication v3.0.0-SNAPSHOT using Java 17.0.6 with PID 1 (/app/target/spring-petclinic-3.0.0-SNAPSHOT.jar started by root in /app)  
2023-03-22T12:33:22.263Z  INFO 1 \--- \[           main\] o.s.s.petclinic.PetClinicApplication     : No active profile set, falling back to 1 default profile: "default"  
2023-03-22T12:33:24.020Z  INFO 1 \--- \[           main\] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.  
2023-03-22T12:33:24.090Z  INFO 1 \--- \[           main\] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 59 ms. Found 2 JPA repository interfaces.  
2023-03-22T12:33:25.031Z  INFO 1 \--- \[           main\] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)  
2023-03-22T12:33:25.060Z  INFO 1 \--- \[           main\] o.apache.catalina.core.StandardService   : Starting service \[Tomcat\]  
2023-03-22T12:33:25.060Z  INFO 1 \--- \[           main\] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: \[Apache Tomcat/10.1.5\]  
2023-03-22T12:33:25.167Z  INFO 1 \--- \[           main\] o.a.c.c.C.\[Tomcat\].\[localhost\].\[/\]       : Initializing Spring embedded WebApplicationContext  
2023-03-22T12:33:25.169Z  INFO 1 \--- \[           main\] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 2784 ms  
2023-03-22T12:33:25.553Z  INFO 1 \--- \[           main\] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 \- Starting...  
2023-03-22T12:33:25.897Z  INFO 1 \--- \[           main\] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection conn0: url=jdbc:h2:mem:de389cf1-5081-414b-99c8-811d092b1701 user=SA  
2023-03-22T12:33:25.910Z  INFO 1 \--- \[           main\] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 \- Start completed.
```

The logs of the collector: `kubectl logs petclinic-<pod_name> -c otc-container`

```
2023\-03\-22T12:33:19.277Z info otlpreceiver@v0.72.0/otlp.go:112 Starting HTTP server {"kind": "receiver", "name": "otlp", "data\_type": "traces", "endpoint": "0.0.0.0:4318"}  
2023\-03\-22T12:33:19.277Z info service/service.go:157 Everything is ready. Begin running and processing data.
```

*For more troubleshooting tips and how to manage your Kubernetes cluster, see the* [*Tools to manage Kubernetes*](https://blog.devops.dev/tools-to-manage-kubernetes-15b675f407d4) *and* [*Lens — a Kubernetes IDE*](https://blog.devops.dev/lens-a-kubernetes-ide-214c1a779c3b)

## Awesome OpenTelemetry

Checkout [Awesome-OpenTelemetry](https://github.com/magsther/awesome-opentelemetry) to quickly get started with OpenTelemetry. This repo contains a big list of helpful resources.

> *An* `*awesome list*` *is a list of awesome things curated by the community. You can read more in the* [*Awesome Manifesto*](https://github.com/sindresorhus/awesome)

## Conclusion

In this post we demonstrated how you can implement traces in your application **without any code changes** by using the **OpenTelemetry Operator.**

**I hope you liked this post. If you found this useful, please hit that clap button and follow me to get more articles on your feed.**