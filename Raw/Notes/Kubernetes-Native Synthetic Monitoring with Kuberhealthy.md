---
title: Kubernetes-Native Synthetic Monitoring with Kuberhealthy
source: https://martinheinz.dev/blog/95
clipped: 2023-09-04
published: 2023-04-17
category: observability
tags:
  - k8s
  - observability
read: false
---

Synthetic monitoring can be a great tool for proactively identifying performance problems, checking availability of servers, monitor DNS resolution and much more.

However, when it comes to synthetic testing, engineers often rely on 3rd party platforms such as *Datadog* or *New Relic* that provide this type of monitoring. If you're running your applications and services on Kubernetes though, you can spin up synthetic monitoring platform yourself using *Kuberhealthy*, and in this article we will take a look at how to deploy it, configure it, create synthetic checks and set up monitoring and alerting, all inside your own cluster.

## Deploy It

Before we go ahead and deploy it, let's first answer one question, what Kuberhealthy actually is?

Kuberhealthy is a CNCF incubator project. It's a Kubernetes operator that provides `KuberhealthyCheck` custom resource, which lets you create builtin or custom synthetic checks that can test whether your cluster, its components or even external services are running as expected.

To deploy it, we first need to satisfy some prerequisites:

Kuberhealthy exposes synthetic check results as Prometheus metrics, so naturally we need Prometheus stack running on our cluster. If you're running applications and services on Kubernetes, chances are you're also running Prometheus stack, but in case you're not or want to follow along in playground cluster, you can use the following:

```
minikube delete && minikube start \
  --kubernetes-version=v1.26.1 \
  --memory=6g \
  --bootstrapper=kubeadm \
  --extra-config=kubelet.authentication-token-webhook=true \
  --extra-config=kubelet.authorization-mode=Webhook \
  --extra-config=scheduler.bind-address=0.0.0.0 \
  --extra-config=controller-manager.bind-address=0.0.0.0

minikube addons disable metrics-server

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack -f values.yaml


kubectl apply -f https://gist.githubusercontent.com/MartinHeinz/6c52f677d7a2fd3a3ff1819190ecd59d/raw/\
    f53a93c425027d9051d1bcdfc3568c6f4a7d9505/kuberhealthy-dashboard-cm.yaml
```

The above creates a new `minikube` cluster with flags necessary for running `kube-prometheus-stack` (Prometheus Operator), which it then also deploys using a Helm Chart. You may also notice that we also provided `values.yaml` during chart installation - this file includes Alertmanager configuration that we will use to send Slack notifications/alerts for failing Kuberhealthy checks later on. This `values.yaml` can be found in this [gist](https://gist.github.com/MartinHeinz/8a4160fd319bd81664ed75bdd0eb722c).

Finally, we also deployed custom Grafana dashboard for Kuberhealthy. This dashboard is available in Kuberhealthy repository and here we deploy it as a *ConfigMap* which will be automatically read by `kube-prometheus-stack`.

With all of this deployed, we can access the Prometheus, Grafana and Alertmanager using:

```
kubectl port-forward -n default svc/monitoring-kube-prometheus-prometheus 9090
kubectl port-forward -n default svc/monitoring-grafana 3000:80  
kubectl port-forward -n default svc/monitoring-kube-prometheus-alertmanager 9093
```

With Prometheus out of the way, let's now deploy Kuberhealthy:

```
helm repo add kuberhealthy https://kuberhealthy.github.io/kuberhealthy/helm-repos
helm install -n kuberhealthy kuberhealthy kuberhealthy/kuberhealthy --create-namespace --values values.yaml
```

Kuberhealthy is also available as a Helm Chart, which makes it easy to deploy. We again provide `values.yaml` to tweak the configuration:

```

prometheus:
  enabled: true

  serviceMonitor:
    enabled: true
    release: monitoring
    namespace: default
    endpoints:
      
      bearerTokenFile: ''
  prometheusRule:
    enabled: true
    release: monitoring
    namespace: default

check:
  daemonset:
    enabled: false
  deployment:
    enabled: false
  dnsInternal:
    enabled: false
```

We use `values.yaml` to enable integration with Prometheus and to specify how `serviceMonitor` and `prometheusRule` provided by Kuberhealthy should be configured so that Prometheus recognizes them. This is done by setting the `release` field to the name of Prometheus deployment (`helm install monitoring ...`).

Kuberhealthy also comes with some builtin checks enabled by default - we disable those for the time being (`check` stanza).

Now we can test if it's running:

```
kubectl port-forward -n kuberhealthy svc/kuberhealthy 8080:80

curl localhost:8080 | jq .
{
  "OK": true,
  "Errors": [],
  "CheckDetails": {},
  "JobDetails": {},
  "CurrentMaster": "kuberhealthy-6b897c89cf-2jpt7"
}

curl 'localhost:8080/metrics'



kuberhealthy_running{current_master="kuberhealthy-6b897c89cf-2jpt7"} 1


kuberhealthy_cluster_state 1
...
```

Generally, you shouldn't need to query these endpoints, because Kuberhealthy metrics are automatically scraped by Prometheus, but it can be handy for doing a manual check or debugging.

Here we `curl` the `kuberhealthy` service which gives us JSON status of the Kuberhealthy cluster. This by default also returns info for all checks in the cluster. If you want to filter by namespace you can instead use e.g. `curl 'localhost:8080/?namespace=default'` for `default` namespace.

In the response from `/metrics` endpoint we see two metrics - `kuberhealthy_cluster_state` and `kuberhealthy_running`. Value of the former provides an *"aggregated status"* of all checks, meaning that if any of checks in the cluster returns 0 (fail), then `kuberhealthy_cluster_state` will also be 0. The latter - `kuberhealthy_running` - tells us whether the Kuberhealthy cluster itself runs.

## Configuring Checks

With deployment and configuration done, we can start deploying our checks:

```
apiVersion: comcast.github.io/v1
kind: KuberhealthyCheck
metadata:
  name: ping-check
  namespace: kuberhealthy
spec:
  runInterval: 30m
  timeout: 10m
  podSpec:
    containers:
      - env:
          - name: CONNECTION_TIMEOUT
            value: "10s"
          - name: CONNECTION_TARGET
            value: "tcp://google.com:443"
        image: kuberhealthy/network-connection-check:v0.2.0
        name: main
```

Each check is an instance of `KuberhealthyCheck` custom resource and each of them specifies `runInterval`, `timeout`, and `podSpec`. These three fields set how often the check should run; after how long it should fail due to timeout; and YAML spec of Pod that will run the check. For the `podSpec`, the important parts are `image` and `env`. The image decides what check we will run, e.g. `kuberhealthy/network-connection-check` performs ping check, while `kuberhealthy/ssl-expiry-check` image will run SSL certificate expiration check. While env variables passed to the Pod (and container) configure how the check will be run, e.g. which host it should query (`CONNECTION_TARGET`).

After deploying this check, a check Pod will be created in `kuberhealthy` namespace. We can check its logs:

```
kubectl logs -n kuberhealthy ping-check-1679831147
time="2023-03-26T11:45:53Z" level=info msg="Found instance namespace: kuberhealthy"
time="2023-03-26T11:45:53Z" level=info msg="Kuberhealthy is located in the kuberhealthy namespace."
time="2023-03-26T11:45:53Z" level=info msg="Check time limit set to: 9m48.53977037s"
time="2023-03-26T11:45:53Z" level=info msg="CONNECTION_TARGET_UNREACHABLE could not be parsed."
time="2023-03-26T11:45:53Z" level=info msg="Running network connection checker"
time="2023-03-26T11:45:53Z" level=info msg="Successfully reported success to Kuberhealthy servers"
time="2023-03-26T11:45:53Z" level=info msg="Done running network connection check for: tcp://google.com:443"
```

And it was successful!

We've now tested a simple ping, what else can we deploy?

```
apiVersion: comcast.github.io/v1
kind: KuberhealthyCheck
metadata:
  name: duration-check
  namespace: kuberhealthy
spec:
  runInterval: 5m
  timeout: 10m
  podSpec:
    containers:
      - name: main
        image: kuberhealthy/http-check:v1.5.0
        imagePullPolicy: IfNotPresent
        env:
          - name: CHECK_URL
            value: "https://httpbin.org/delay/9"
          - name: COUNT
            value: "5"
          - name: SECONDS
            value: "1"
          - name: REQUEST_TYPE
            value: "GET"
          - name: PASSING
            value: "80"
---
apiVersion: comcast.github.io/v1
kind: KuberhealthyCheck
metadata:
  name: http-content-check
  namespace: kuberhealthy
spec:
  runInterval: 60s
  timeout: 2m
  podSpec:
    containers:
      - image: kuberhealthy/http-content-check:v1.5.0
        imagePullPolicy: IfNotPresent
        name: main
        env:
          - name: "TARGET_URL"
            value: "https://httpbin.org/anything/whatever"
          - name: "TARGET_STRING"
            value: "whatever"
          - name: "TIMEOUT_DURATION"
            value: "30s"
```

Here we use 2 more builtin checks - `http-check` and `http-content-check` - which serve similar use case as the ping check. First one lets you do HTTP request(s) to specified URL, with customizable number of request, expected passing percentage, timeout for individual requests, as well as request type.

While the other lets you check whether some value (`TARGET_STRING`) is present in response from a requested URL.

Another useful synthetic test is to check whether SSL certificate of a website is about to expire, there's builtin check for that too:

```
apiVersion: comcast.github.io/v1
kind: KuberhealthyCheck
metadata:
  name: website-ssl-expiry-30d
  namespace: kuberhealthy
spec:
  runInterval: 24h
  timeout: 15m
  podSpec:
    containers:
      - env:
          - name: DOMAIN_NAME
            value: "martinheinz.dev"
          - name: PORT
            value: "443"
          - name: DAYS
            value: "30"
          - name: INSECURE
            value: "false"  
        image: kuberhealthy/ssl-expiry-check:v3.2.0
        imagePullPolicy: IfNotPresent
        name: main
```

There are a couple more builtin checks you can use, but I don't want to go over every single one of them, instead I would recommend you check out [check registry](https://github.com/kuberhealthy/kuberhealthy/blob/master/docs/CHECKS_REGISTRY.md) which includes list of all the available builtin checks along with examples.

## Monitoring

With all the checks in place, it's time to start monitoring their results. To do so, we will write some *PromQL* queries. Besides the `kuberhealthy_cluster_state` and `kuberhealthy_running` mentioned earlier, Kuberhealthy provides `kuberhealthy_check{check='namespace/check-name'}` and `kuberhealthy_check_duration_seconds{check='namespace/check-name'}` for each check. We will use those to build our monitoring rules and alerts.

![[Raw/Notes/Raw/Media/Resources/e675622045af0d773ec6d1cd0ea293b4_MD5.webp]]

For availability reasons, Kuberhealthy runs multiple replicas of the operator, which means that we will get multiple results (series) - one for each operator Pod - for each queried check.

To avoid that, we will only query results (series) from current master Pod in the Kuberhealthy cluster which provides the authoritative data. To do so we will use following query:

```
label_replace(kuberhealthy_check{check="kuberhealthy/ping-check"}, "current_master", "$1", "pod", "(.+)") \
    * on (current_master) group_left() \
    topk(1, kuberhealthy_running{}) < 1
```

Let's explain what's happening here - `kuberhealthy_running` metric has `current_master` label that refers to the master Pod in Kuberhealthy cluster, while `kuberhealthy_check` metric has `pod` label which refers to pod from which it originates. We only want to query `kuberhealthy_check` metrics that have `pod` label value that equals to `current_master` label value of `kuberhealthy_running` metric, but to be able to match 2 metrics against each other they need to have a label with the same name. So, we use `label_replace` function to replace the `pod` label in `kuberhealthy_check` metric with `current_master`. Now both metrics have `current_master` label and we can query only the ones that match.

Alternatively, instead of taking results only from current master, you can simply use `topk(1, kuberhealthy_check{check="kuberhealthy/ping-check"})` to grab the first series, which should work fine *almost* always, assuming the data is consistent across all cluster Pods.

Now, when you query any of the above `kuberhealthy_*` metrics, you will notice that they have a `status` label with value of `0`/`1`. While a metric value obviously describes whether the check succeeded or failed, the `status` label describes whether there was an error (`0`) or not (`1`). This is important for monitoring, because there is a big difference between a failing check and a broken one.

Now we know what queries we can build, but we need to create *PrometheusRule(s)* from them to be able to set up monitoring/alerting:

```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: synthetics
  namespace: default
  labels:
    prometheus: prometheus
    release: monitoring
spec:
  groups:
    - name: synthetics
      rules:
        - alert: PingFailed
          expr: >
            label_replace(kuberhealthy_check{check="kuberhealthy/ping-check"}, "current_master", "$1", "pod", "(.+)")
            * on (current_master) group_left() topk(1, kuberhealthy_running{}) < 1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: HTTP Ping failed
            description: "Kuberhealthy was not able to reach tcp://google.com:443"
        - alert: SslExpiryLessThan30d
          expr: topk(1, kuberhealthy_check{check='kuberhealthy/website-ssl-expiry-30d'}) < 1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: Server certificate will expire in less than 30 days
            description: "Server certificate will expire in less than 30 days"
        - alert: DurationCheckFailed
          expr: >
            avg without (endpoint, container, service, current_master, exported_namespace, job)
            (label_replace(kuberhealthy_check_duration_seconds{check='kuberhealthy/duration-check'}, "current_master", "$1", "pod", "(.+)")
            * on (current_master) group_left() topk(1, kuberhealthy_running{})) > 50
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: Check taking more than 50s to complete
            description: Check taking more than 50s to complete ().
```

The important parts above really are just `expr` stanzas, in the first one we use the query that takes data from Kuberhealthy master Pod and expects the value to be `1`, otherwise the rule/alerts gets triggered (after 5 minutes). In the second one we take the simpler approach and just grab the first value using `topk`.

In the third check we use `kuberhealthy_check_duration_seconds` metric and compute the average time it takes to run the check and we have it fail if it's more than 50 seconds. Word of caution for the *"duration"* metrics though - they describe the length of lifetime of whole check Pod, not the individual check attempts - you should take that into consideration when deciding the threshold for rule success/fail.

![[Raw/Notes/Raw/Media/Resources/75af22eb9f7ba62a2d4721f269967a7a_MD5.webp]]

Finally, if you used the provided Alertmanager configuration shown in the beginning, you should be able to receive Slack alerts such as:

![[Raw/Notes/Raw/Media/Resources/951d22fa27c6f502549f62e70ac22564_MD5.webp]]

In addition to these binary (`true`/`false`) rules, [Kuberhealthy docs](https://github.com/kuberhealthy/kuberhealthy/blob/master/docs/K8s-KPIs-with-Kuberhealthy.md#creating-key-performance-indicators) provide examples of calculating availability, utilization and latency from the available metrics.

## Writing Custom Checks

So far we only used the builtin checks which to be honest will be sufficient for most of the tests. However, it's possible to built your own. Such custom check could - for example - implement checking a service that requires authentication or database-native check that validates if it's possible to connect/run query against DB.

Building custom check would warrant a separate article, so instead, to avoid making this article way too long, I will just leave you with [docs link](https://github.com/kuberhealthy/kuberhealthy/blob/master/docs/CHECK_CREATION.md) which explains how you can build a check image in language of your choice.

Also, if you want some inspiration or a starting point, I have [a repository](https://github.com/MartinHeinz/kuberhealthy-custom-checks) with custom check(s), such as `jq-check` which uses `jq` query to check whether a HTTP JSON response contains expected data/value.

## Conclusion

If you're running applications and services on Kubernetes, chances are you're also running Prometheus stack and relying on application metrics for monitoring. While this is a good practice, it's not necessarily sufficient, as metrics aren't suitable for monitoring everything, such as SSL expiration or server/database connectivity.

Tools like Kuberhealthy are a great for filling these gaps in monitoring, while allowing you to use the same, familiar interface - that is - Kubernetes and Prometheus.

Finally, while Kuberhealthy runs on Kubernetes and aims at testing Kubernetes cluster itself, it is not confined only to the cluster. It can be also used to test external resources, such as databases or services in the Cloud, or legacy applications that don't expose metrics.