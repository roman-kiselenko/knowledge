---
title: "Kubernetes: Dynamic Admission Control"
source: https://dev.to/gkampitakis/kubernetes-dynamic-admission-control-1f9p
clipped: 2023-09-09
published: 2023-05-13
category: development
tags:
  - development
  - k8s
  - operators
read: true
---


This is a demo for [Kuberentes Dynamic Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/), exploring various concepts, capabilities and constraints, it's following [A Guide to Kubernetes Admission Controllers](https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/) which contains a lot of information around Admission Controllers and Kubernetes.

## [](#what-is-dynamic-admission-control)What is Dynamic Admission Control

Borrowing the definion from the [Kubernetes page](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)

> In addition to [compiled-in admission plugins](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/), admission plugins can be developed as extensions and run as webhooks configured at runtime.
> 
> Admission webhooks are HTTP callbacks that receive admission requests and do something with them. You can define two types of admission webhooks, [validating admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook) and [mutating admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook).

You can [read some of the use cases](https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/#why-do-i-need-admission-controllers) for admission controllers.

In the demo we will focus on validating admission webhooks, but most of the principles apply on mutating as well.

## [](#demo)Demo

We are going to run the demo locally on [minikube](https://minikube.sigs.k8s.io/docs/start/).

### [](#prerequisites)Prerequisites

You will need some tools installed for following along

-   Clone code from [Github](https://github.com/gkampitakis/k8s-dac-demo)
-   [minikube](https://minikube.sigs.k8s.io/docs/start/)
-   [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
-   [docker](https://docs.docker.com/engine/)
-   [go](https://go.dev/doc/install) (*optional if you want to play with webhook server code*)

### [](#setting-up-minikube)Setting up minikube

We are going to spin up a minikube cluster by running  

```
# start kubernetes cluster locally
minikube start

# enable needed addons
minikube addons enable metrics-server
minikube addons enable registry
```


We are setting up a local image registry so we don't rely on pushing/pulling the webhook server container image on a remote registry e.g. [docker hub](https://hub.docker.com/).  

```
# run local registry
docker run --rm -it --network=host alpine ash -c "apk add socat && socat TCP-LISTEN:5000,reuseaddr,fork TCP:$(minikube ip):5000"
```


All the components for running and testing the webhook now should be ready.

### [](#building-and-deploying)Building and Deploying

The next step is to build the Webhook Docker image and deploy it alongside with the Admission Webhook configuration.

1.  You will need to build and push the container to your local registry by running:

```
make docker-image-local
```


1.  then you are ready to deploy the Validating Admision Webhook in your local minikube cluster with:

```
make deploy
```


The `make deploy` command does a couple of actions

1.  Generates TLS Certificates, since a webhook must be served via HTTPS, we need proper certificates for the server. \[ [More](https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/#tls-certificates) \]
2.  Creates a dedicated namespace for putting the Webhook Server `Deployment` and the `ValidatingWebhookConfiguration`. This useful if you want to skip your Webhook from "validating itself".
3.  Creates a Kubernetes TLS secret with the generated key and certificate so the Webhook server can access.
4.  Creates a `Service` for reaching the Webhook Server `Pods` at port `443`.
5.  Creates the Webhook Server `Deployment`.
6.  Finally after everything is in place creates the `ValidatingWebhookConfiguration`.

### [](#the-webhook-server)The Webhook Server

The Webhook Server we are deploying is a simple Go HTTP server that accepts `AdmissionReview` POST requests at `/validate`, verifys the payload is correct.

Then checks the resource passed for review, in the demo case we are only care about `POD` creation, if it contains a `team` label set up. You can configure rules for what review requests are directed to your Webhook in `ValidatingWebhookConfiguration` object. More details on [Matching requests: rules](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#matching-requests-rules).

If this label `team: <value>` is missing the Webhook Server will return either a warning that the lable is missing or a failure that prevents the `POD` from scheduling depending on the `ALLOW_SCHEDULING` env var.  

```go
if _, exists := pod.Labels["team"]; !exists {
    admissionReviewResponse.Response.Allowed = allowScheduling
    if allowScheduling {
        admissionReviewResponse.Response.Warnings = []string{
            "Team label not set on pod",
        }
    }
    admissionReviewResponse.Response.Result = &metav1.Status{
        Status:  "Failure",
        Message: "Team label not set on pod",
    }

    log.Printf("Team label not set on pod: %s\n", pod.Name)
}
```


In the warning case you will see this message  

```
Warning: Team label not set on pod
```


Also it's a good practise to skip validating namespaces that are used by Kubernetes, e.g. `kube-public` and `kube-system`, as you might cause a disruption of you cluster if you deny scheduling of resources there.

You can experiment with the Webhook Server [code](https://github.com/gkampitakis/k8s-dac-demo/blob/main/webhook-server/main.go) do changes and redeploy it. You can build a new docker image and push it to the local registry we set up earlier with  

```
make docker-image-local
```


and then restart the deployment for creating new `PODS` with the new image  

```
kubectl rollout restart deploy/webhook-server
```


### [](#verifying-everything-works)Verifying everything Works

Okay let's check that everything works and see the Validating Webhook in practise.

First we can check that the Webhook Server is scheduled and healty. You can do it with  

```
kubectl get pods -n webhook-demo
```


and the output should be similar to this  

```
NAME                              READY   STATUS    RESTARTS   AGE
webhook-server-79c79b5877-d624n   1/1     Running   0          11m
webhook-server-79c79b5877-dpf4b   1/1     Running   0          11m
```


Then if we try to schedule [a simple](https://github.com/gkampitakis/k8s-dac-demo/blob/main/deployment/busy-box.yaml) `POD` it should fail to schedule with the error  

```
kubectl apply -f deployment/busy-box.yaml

Error from server: error when creating "deployment/busy-box.yaml": admission webhook "webhook-server.webhook-demo.svc" denied the request: Team label not set on pod
```


You can add a team label in the pod spec and try again successfully.

## [](#observations)Observations

### [](#namespace-selector)Namespace Selector

You might have noticed the [code](https://github.com/gkampitakis/k8s-dac-demo/blob/1b5648cc00b33257cd42efd5343a1c3fd2cc9a4e/webhook-server/main.go#L164-L172) in the Webhook Server contains logic for skipping validation for a list of namespaces.

But `ValidatingWebhookConfiguration` supports natively limiting requests that reach to your Webhook Server depending on [namespaceSelector](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#matching-requests-namespaceselector).

I couldn't find a way to whitelist namespaces by their name. For using namespaceSelector you can add this to `ValidatingWebhookConfiguration`.  

```
namespaceSelector:
  matchExpressions:
    - key: webhook-skip
      operator: DoesNotExist
```


and then mark all namespaces you want to skip with this label  

```
kubectl label ns webhook-demo webhook-skip=true
```


### [](#reliability-and-availability)Reliability and availability

Dynamic Admission Controller can be on the critical path, depending how you your Webhook configured e.g. can block various actions on Kubernetes resources.

This is a non exhaustive list of things you can do to increase the reliability and the availability of your admission controller.

-   The Dynamic Admission Controller is backed by an HTTP Server and a `Deployment`. This means all concepts for making a stateless deployment more reliable in Kubernetes ( *if it is hosted in Kubernetes* )
    -   Use load balancer in front of your Webhook server that provides [High Availability](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#availability)
    -   Use [Rolling Updates](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/#:~:text=with%20rolling%20updates.-,Rolling%20updates%20allow%20Deployments%27%20update%20to%20take%20place%20with%20zero%20downtime%20by%20incrementally%20updating%20Pods%20instances%20with%20new%20ones.,-The%20new%20Pods)
    -   [Pod Disruption Budget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
    -   Rollback to an earlier Deployment if the current state is not stable
    -   Autoscalling for matching increasing demand \[ [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) \]
-   [Avoiding deadlock in self hosted webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#avoiding-deadlocks-in-self-hosted-webhooks)
-   [Avoiding operating on the kube-system namespace](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#avoiding-operating-on-the-kube-system-namespace)
-   It is recommended that admission webhooks should evaluate as quickly as possible (typically in milliseconds), since they add to API request latency. It is encouraged to use a small [timeout for webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#timeouts).
-   Setting [Failure Policy](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#failure-policy) depending on your needs.
-   And last but not least always monitoring and observing your Webhooh Server status and the Kube API server status. The API Server exposes Prometheus metrics from the `/metrics` endpoint e.g. `apiserver_admission_webhook_admission_duration_seconds_sum`,`apiserver_admission_webhook_rejection_count`.

## [](#summary)Summary

Dynamic Admission Controllers provide great benefits for security and governance across your Kubernetes Cluster. This demo can help you get started building and deploying a Validating Admission Webhook.

#### [](#resources)Resources

-   [Dynamic Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers)
-   [A Guide to Kubernetes Admission Controllers](https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/)
-   [Github repository](https://github.com/gkampitakis/k8s-dac-demo)