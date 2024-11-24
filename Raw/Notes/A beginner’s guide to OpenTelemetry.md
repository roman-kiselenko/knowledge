---
title: A beginner’s guide to OpenTelemetry
source: https://faun.pub/opentelemetry-d71d369c83d7
clipped: 2023-09-23
published: 
category: observability
tags:
  - k8s
  - observability
read: true
---

[OpenTelemetry](https://opentelemetry.io/) is a collection of tools, **APIs**, and **SDKs**.

Let’s go through what you need to know to get started with [OpenTelemetry](https://opentelemetry.io/).

![[Raw/Media/Resources/377e56f62851a860a6c8ddc4d2f38a06_MD5.png]]

OpenTelemetry solution

1.  **Instrument** the application using OpenTelemetry SDK.
2.  Traces are sent to a (collector) **agent**.
3.  The collector **exposes** ports **4317**(gRPC) and/or **4318**(http) using **OTLP.**
4.  Traces are then **exported** (in this case to [**Jaeger**](https://www.jaegertracing.io/)) and stores the traces on the **backend**
5.  The **Jaeger Query** retrieves the traces and expose them to the Jaeger UI.
6.  **Jaeger UI** provides a web based user interface that can be used to analyse traces.
7.  The **storage backend** can be [Jaeger](https://www.jaegertracing.io/), [Zipkin](https://zipkin.io/), [Elastic](https://www.elastic.co/), [Cassandra](https://cassandra.apache.org/_/index.html), [Tempo](https://grafana.com/oss/tempo/), [Splunk](https://www.splunk.com/), a vendor ([Honeycomb](https://www.honeycomb.io/), [Lightstep](https://lightstep.com/), [Signoz](https://signoz.io/), [Logz.io](https://logz.io/), [Aspecto](https://www.aspecto.io/) etc..).

Defines how OpenTelemetry is used.

Defines the specific implementation of the API for a language.

As you can see from the image (at the top), in order to get trace data, we first need to **instrument** the application. To collect the trace data, we can use the **OpenTelemetry SDK.**

The trace data can be generated using either **automatic** or **manual (**or a mix**)** instrumentation.

To instrument your application with [OpenTelemetry](https://opentelemetry.io/), go to the OpenTelemetry [repository](https://github.com/open-telemetry), and pick the language for your application and follow the instructions.

## Auto Instrumentation

*One of the best ways to instrument applications is to use OpenTelemetry automatic instrumentation (auto-instrumentation). This approach is simple, easy, and doesn’t require many code changes.*

Using auto-instrumentation libraries, means that you **don’t need to** write code for the trace information to be collected. ​ ​In fact, OpenTelemetry offers an API and SDK that allows for easy `bootstrapping` of distributed tracing into your software.

> This is good to know if don’t have the necessary knowledge (or time) to create a tracing framework tailored for your application.

On [OpenTelemetry Registry](https://opentelemetry.io/registry/) you can search for libraries, plugins, integrations, and other useful tools for extending OpenTelemetry.

For example, running the following command will automatically instrument your python code.

`opentelemetry-instrument python app.py`

When you use auto instrumentation, a predefined sets off spans will be created for you and populated with relevant attributes.

## Manual Instrumentation

Manual instrumentation is when you write specific code for your application. *It’s the process of adding observability code to your application.* This can more effectively suit your needs. For example, you can add attributes and events.

Once you have collected trace data, you need to **send** it somewhere.

*OpenTelemetry Protocol (OTLP) specification describes the encoding, transport, and delivery mechanism of telemetry data between telemetry sources, intermediate nodes such as collectors and telemetry backends.* [source](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/protocol/otlp.md)

The OTLP protocol describes how to encode and transmit telemetry data, which makes it a natural choice for data transport. Each language SDK provides an OTLP exporter you can configure to export data over OTLP. The OpenTelemetry SDK then transforms events into OTLP data.

The data from your instrumented application can be sent to an [OpenTelemetry](https://opentelemetry.io/) collector.

The collector is a component of [OpenTelemetry](https://opentelemetry.io/) that **collects** trace data (spans, metrics, logs etc) **process** (pre-processes data), and **exports** the data (sends it off to a backend that you want to talk to).

The [OpenTelemetry](https://opentelemetry.io/) **collector** can receive telemetry data in **multiple formats**.

![[Raw/Media/Resources/7a33d1321b49ee0f2ce0af96f2c004ae_MD5.png]]

## Agent vs Gateway

The **collector** can be setup as an **agent** or as a **gateway**.

We usually first send traces to a (collector) **agent**. This (collector) agent handles the trace data from the instrumented application.

The (collector) agent can **offload responsibilities** that the client instrumentation otherwise need to handle. This includes **batching**, **retry**, **encryption**, **compression** and more.

> You can also perform **sampling** here depending on the amount of traces/traffic you want to send. (*ie. take only 10% of the traces*).

You need to configure **receivers** (*how data gets into the collector*), which then **transform** the data (**process)** before sending it to one or more backends using **exporters.**

## Receivers

Here we configure a **receiver** (on the collector) that accepts OTLP data on port **4317** (gRPC). (You also need to configure your application to export over OTLP to use it)

```yaml
otlp:  
   protocols:  
      grpc:  
        endpoint: "0.0.0.0:4317"

```
> In our [OpenTelemetry on Kubernetes](https://medium.com/@magstherdev/opentelemetry-on-kubernetes-c167f024b35f) article, we will talk about this in more detail and also show how you can deploy this on a local Kubernetes cluster.

## Exporters

Once you’ve instrumented your code, you need to get the data out in order to do anything useful with it. [OpenTelemetry](https://opentelemetry.io/) comes with a variety of **exporters**.

The exporters **converts** OpenTelemetry protocol (**OTLP**) formatted data to their respective predefined back-end format and exports this data to be interpreted by the back-end or system.

For example, for metrics, there is a [Prometheus](https://opentelemetry.io/docs/instrumentation/js/exporters/#prometheus) exporter that allows sending metrics so that Prometheus can consume it.

A common scenario (especially during testing), is to export all data directly to [Jaeger](https://www.jaegertracing.io/) using the **jaeger-exporter**.

> In our [OpenTelemetry on Kubernetes](https://medium.com/@magstherdev/opentelemetry-on-kubernetes-c167f024b35f) article, we will talk about this in more detail and also show how you can deploy this on a local Kubernetes cluster.

You can read more about exporters [here](https://opentelemetry.io/docs/instrumentation/js/exporters/)

## Storage Backends

> *It’s important to know that the OpenTelemetry collector does not provides their own backend.*

The **storage backend** can be , , , , , [New Relic](https://newrelic.com/), [Zipkin](https://zipkin.io/), [Elastic](https://www.elastic.co/), [Cassandra](https://cassandra.apache.org/_/index.html), [Tempo](https://grafana.com/oss/tempo/), [Splunk](https://www.splunk.com/).

For a full list of storage alternatives, please checkout the [Awesome OpenTelemetry](https://github.com/magsther/awesome-opentelemetry#storage) repository.

## OpenTelemetry on Kubernetes

This OpenTelemetry [repo](https://github.com/open-telemetry/opentelemetry-go/tree/main/example/otel-collector) provides a complete demo on how you can deploy OpenTelemetry on Kubernetes.

Please check this [article](https://medium.com/@magstherdev/opentelemetry-on-kubernetes-c167f024b35f) on how to deploy this on a local Kubernetes cluster.

## Awesome OpenTelemetry

Checkout [Awesome-OpenTelemetry](https://github.com/magsther/awesome-opentelemetry) to quickly get started with **OpenTelemetry.** This repo contains a big list of helpful resources.

> *An* `*awesome list*` *is a list of awesome things curated by the community. You can read more in the* [*Awesome Manifesto*](https://github.com/sindresorhus/awesome)

I hope you liked this article about [OpenTelemetry](https://opentelemetry.io/). If you found this useful, please hit that **clap** button and **follow me** to get more articles on your feed.
