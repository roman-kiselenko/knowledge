---
title: Kubernetes Event Aggregation and Spam Filtering in client-go
source: https://dev.to/shuheiktgw/kubernetes-event-aggregation-and-spam-filtering-in-client-go-25mn?s=35
clipped: 2023-09-18
published: 2023-04-18
category: k8s
tags:
  - development
read: true
---

## [](#tldr)TL;DR

-   [clint-go](https://github.com/kubernetes/client-go) has features for event aggregation, which groups similar events into one, and spam filtering, which applies rate limits to events.
-   [Event aggregation](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L242) uses an aggregation key generated by the [EventAggregatorByReasonFunc](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L165) as a key, and if the same event is published 10 or more times within 10 minutes, instead of posting a new event, update the existing to save etcd capacity.
-   [Spam filtering](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L129) performs using the value generated by the [getSpamKey function](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L71) as a key. The spam filter uses a rate limiter based on the token bucket algorithm, with an initial 25 tokens and a refill rate of 1 token per 5 minutes. If an event is published beyond the limit, the event is discarded.

## [](#background)Background

When I wanted to create a CustomController that detects the completion of a child Job of a CronJob and performs an action, I noticed that CronJobController (v2) published an Event called [SawCompletedJob](https://github.com/kubernetes/kubernetes/blob/1b825c179baad213bb570787f719d74211be3bcf/pkg/controller/cronjob/cronjob_controllerv2.go#L452) when the child Job completed. I thought, "Oh, I can simply use the Event as a cue."

However, when I created a CronJob that ran once a minute in a local cluster (kind), I discovered that a SawCompletedJob event became unobservable from the middle of the run (about the 10th run). Specifically, the Events looked like the ones below.  

```
$ kubectl alpha events --for cronjob/hello
...
8m29s (x2 over 8m29s)   Normal   SawCompletedJob    CronJob/hello   Saw completed job: hello-28023907, status: Complete
8m29s                   Normal   SuccessfulDelete   CronJob/hello   Deleted job hello-28023904
7m35s                   Normal   SuccessfulCreate   CronJob/hello   Created job hello-28023908
7m28s (x2 over 7m28s)   Normal   SawCompletedJob    CronJob/hello   Saw completed job: hello-28023908, status: Complete
7m28s                   Normal   SuccessfulDelete   CronJob/hello   Deleted job hello-28023905
6m35s                   Normal   SuccessfulCreate   CronJob/hello   Created job hello-28023909
6m28s                   Normal   SawCompletedJob    CronJob/hello   Saw completed job: hello-28023909, status: Complete
2m35s (x3 over 4m35s)   Normal   SuccessfulCreate   CronJob/hello   (combined from similar events): Created job hello-28023913
```

Enter fullscreen mode Exit fullscreen mode

SawCompletedJob events were published until 6m28s, but they became unobservable after that. SuccessfulCreate events were published in an aggregated form, but we cannot see all of them. By the way, all events from child Jobs and child Pods are observable.  

```
$ kubectl alpha events

4m18s                 Normal    Scheduled                 Pod/hello-28023914-frb94   Successfully assigned default/hello-28023914-frb94 to kind-control-plane
4m11s                 Normal    Completed                 Job/hello-28023914         Job completed
3m18s                 Normal    Started                   Pod/hello-28023915-5fsh5   Started container hello
3m18s                 Normal    SuccessfulCreate          Job/hello-28023915         Created pod: hello-28023915-5fsh5
3m18s                 Normal    Created                   Pod/hello-28023915-5fsh5   Created container hello
3m18s                 Normal    Pulled                    Pod/hello-28023915-5fsh5   Container image "busybox:1.28" already present on machine
3m18s                 Normal    Scheduled                 Pod/hello-28023915-5fsh5   Successfully assigned default/hello-28023915-5fsh5 to kind-control-plane
3m11s                 Normal    Completed                 Job/hello-28023915         Job completed
2m18s                 Normal    Started                   Pod/hello-28023916-qbqqk   Started container hello
2m18s                 Normal    Pulled                    Pod/hello-28023916-qbqqk   Container image "busybox:1.28" already present on machine
2m18s                 Normal    Created                   Pod/hello-28023916-qbqqk   Created container hello
2m18s                 Normal    SuccessfulCreate          Job/hello-28023916         Created pod: hello-28023916-qbqqk
2m18s                 Normal    Scheduled                 Pod/hello-28023916-qbqqk   Successfully assigned default/hello-28023916-qbqqk to kind-control-plane
2m11s                 Normal    Completed                 Job/hello-28023916         Job completed
78s                   Normal    SuccessfulCreate          Job/hello-28023917         Created pod: hello-28023917-kpxvn
78s                   Normal    Created                   Pod/hello-28023917-kpxvn   Created container hello
78s                   Normal    Pulled                    Pod/hello-28023917-kpxvn   Container image "busybox:1.28" already present on machine
78s                   Normal    Started                   Pod/hello-28023917-kpxvn   Started container hello
78s                   Normal    Scheduled                 Pod/hello-28023917-kpxvn   Successfully assigned default/hello-28023917-kpxvn to kind-control-plane
71s                   Normal    Completed                 Job/hello-28023917         Job completed
18s (x8 over 7m18s)   Normal    SuccessfulCreate          CronJob/hello              (combined from similar events): Created job hello-28023918
18s                   Normal    Started                   Pod/hello-28023918-grvbz   Started container hello
18s                   Normal    Created                   Pod/hello-28023918-grvbz   Created container hello
18s                   Normal    Pulled                    Pod/hello-28023918-grvbz   Container image "busybox:1.28" already present on machine
18s                   Normal    SuccessfulCreate          Job/hello-28023918         Created pod: hello-28023918-grvbz
18s                   Normal    Scheduled                 Pod/hello-28023918-grvbz   Successfully assigned default/hello-28023918-grvbz to kind-control-plane
11s                   Normal    Completed                 Job/hello-28023918         Job completed
```

Enter fullscreen mode Exit fullscreen mode

As I couldn't find any specific description of this behavior in the Kubernetes official documentation, I investigated it by reading the source code to figure out what was happening.

## [](#kubernetes-event-publication-flow)Kubernetes Event Publication Flow

client-go sends Kubernetes Events to kube-apiserver through and etcd stores them. client-go is responsible for event aggregation and spam filtering, but the flow of client-go sending Events to kube-apiserver is quite complex, so we will explain it first. Note that we will use the SawCompleteJob Event of CronJobController as an example, but please note that the details may vary on each controller.

[![[Raw/Media/Resources/fa56e894edcfcabe6f4a3f7c746dcda7_MD5.jpg]]](https://res.cloudinary.com/practicaldev/image/fetch/s--q4mYuLse--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_800/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/sacw777gmi2ghdt354f8.png)

1.  CronJobController publishes an Event via the Recorder's [Eventf method](https://github.com/kubernetes/kubernetes/blob/1b825c179baad213bb570787f719d74211be3bcf/pkg/controller/cronjob/cronjob_controllerv2.go#L452). The method internally calls the Broadcaster's [ActionOrDrop method](https://github.com/kubernetes/apimachinery/blob/40ea93bb2f462f29e5af2018b7380683788a20c2/pkg/watch/mux.go#L231) and sends the Event to the incoming channel.
2.  The Event in the incoming channel is retrieved by [the loop goroutine](https://github.com/kubernetes/apimachinery/blob/40ea93bb2f462f29e5af2018b7380683788a20c2/pkg/watch/mux.go#L264-L277) and [forwarded to each Watcher's result channel](https://github.com/kubernetes/apimachinery/blob/40ea93bb2f462f29e5af2018b7380683788a20c2/pkg/watch/mux.go#L281-L288).
3.  The Event Watcher goroutine calls [the eventHandler](https://github.com/kubernetes/client-go/blob/f6a5a1f1391b9b6ceaaa499bd7cebf508d5e02a6/tools/record/event.go#L327) for the Event received from the result channel. The eventHandler calls [recordToSink in the eventBroadcasterImpl](https://github.com/kubernetes/client-go/blob/f6a5a1f1391b9b6ceaaa499bd7cebf508d5e02a6/tools/record/event.go#L199-L201), where EventCorrelator performs event aggregation and spam filtering and then calls [recordEvent](https://github.com/kubernetes/client-go/blob/f6a5a1f1391b9b6ceaaa499bd7cebf508d5e02a6/tools/record/event.go#L251) to post an Event (or update the Event if it has been aggregated).

### [](#note-starting-loop-goroutine)Note: Starting loop goroutine

The loop goroutine starts through [the NewLongQueueBroadcaster function](https://github.com/kubernetes/apimachinery/blob/40ea93bb2f462f29e5af2018b7380683788a20c2/pkg/watch/mux.go#L84-L95), which is called through [the NewBroadcaster function](https://github.com/kubernetes/client-go/blob/f6a5a1f1391b9b6ceaaa499bd7cebf508d5e02a6/tools/record/event.go#L161-L164). The NewBroadcaster function is called in [the NewControllerV2 function](https://github.com/kubernetes/kubernetes/blob/1b825c179baad213bb570787f719d74211be3bcf/pkg/controller/cronjob/cronjob_controllerv2.go#L85).

### [](#note-starting-event-watcher)Note: Starting Event Watcher

The CronJobController calls [the StartRecordingToSink method](https://github.com/kubernetes/kubernetes/blob/1b825c179baad213bb570787f719d74211be3bcf/pkg/controller/cronjob/cronjob_controllerv2.go#L133) of eventBroadcasterImpl, which starts the Event Watcher from [the StartEventWatcher method](https://github.com/kubernetes/client-go/blob/f6a5a1f1391b9b6ceaaa499bd7cebf508d5e02a6/tools/record/event.go#L198-L201). The StartEventWatcher method initializes and registers the Watcher through [the Watch method](https://github.com/kubernetes/apimachinery/blob/40ea93bb2f462f29e5af2018b7380683788a20c2/pkg/watch/mux.go#L141-L158) of the Broadcaster. What I found interesting is that [the registration process of the Watcher itself](https://github.com/kubernetes/apimachinery/blob/40ea93bb2f462f29e5af2018b7380683788a20c2/pkg/watch/mux.go#L144-L152) is sent to the incoming channel, and [the loop goroutine executes it](https://github.com/kubernetes/apimachinery/blob/40ea93bb2f462f29e5af2018b7380683788a20c2/pkg/watch/mux.go#L269-L272), making the events published before the start of the Watcher invisible to it ([the comment](https://github.com/kubernetes/apimachinery/blob/40ea93bb2f462f29e5af2018b7380683788a20c2/pkg/watch/mux.go#L113-L116) calls it as a "terrible hack" though).

## [](#eventcorrelator)EventCorrelator

EventCorrelator implements the core logic for Kubernetes Event aggregation and spam filtering. EventCorrelator is initialized through the NewEventCorrelatorWithOptions function called in [the StartRecordingToSink method](https://github.com/kubernetes/client-go/blob/f6a5a1f1391b9b6ceaaa499bd7cebf508d5e02a6/tools/record/event.go#L197) of eventBroadcasterImpl. Note that e.options is empty, so the Controller uses [the default values](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L457). eventBroadcasterImpl's [recordToSink method](https://github.com/kubernetes/client-go/blob/f6a5a1f1391b9b6ceaaa499bd7cebf508d5e02a6/tools/record/event.go#L209) calls [EventCorrelate method](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L510), which aggregates Events and applies tge spam filter.

### [](#aggregation)Aggregation

The EventCorrelator's [EventAggregate method](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L242) and [the eventObserve method](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L327) of eventLogger are used for Event aggregation. The source code has detailed comments, so it's recommended to refer to it directly for more information, but here's a brief overview of the process:

1.  Calculate aggregationKey and localKey using [EventAggregatorByReasonFunc](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L165). aggregationKey consists of event.Source, event.InvolvedObject, event.Type, event.Reason, event.ReportingController, and event.ReportingInstance, while localKey is event.Message.
2.  Search the cache of EventAggregator using the the aggregationKey. The cache value is [aggregateRecord](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L227). If the number of the localKeys within the maxIntervalInSeconds ([default 600 seconds](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L42)) is greater than or equal to the maxEvents ([default 10](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L41)), return aggregationKey as the key, otherwise return [the eventKey](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L246) as the key. Note that the eventKey should be unique to each Event.
3.  Call [the eventObserve method](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L327) of eventLogger with the key returned from the EventCorrelate method, and search the cache of eventLogger. The cache value is [eventLog](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L300). If it hits the cache (i.e., the key is aggregationKey), compute the patch to update the Event.

### [](#spam-filtering)Spam Filtering

For spam filtering, [the filterFunc of EventCorrelator is called](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L516) to apply it. The actual implementation of filterFunc is [the Filter method](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L129) of EventSourceObjectSpamFilter. Again, it's recommended to refer to the source code for details, but here's a brief overview of the process:

1.  Calculate the eventKey from the Event using [the getSpamKey function](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L71).
2.  Search the cache of EventSourceObjectSpamFilter using the eventKey. The cache value is [spamRecord](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L123), which contains [the rate limiter](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L145). [The default values for qps and burst of the rate limiter](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L441) are [1/300](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L49) and [25](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L48), respectively. According to [the comment](https://github.com/kubernetes/client-go/blob/b9fa896d5ded5a18c5c84b3f6e80959c303dd05c/util/flowcontrol/throttle.go#L61-L62), the rate limiter uses the token bucket algorithm, so there are initially 25 tokens, and then one token is refilled every 5 minutes.
3.  Call [the TryAccept method](https://github.com/kubernetes/client-go/blob/b9fa896d5ded5a18c5c84b3f6e80959c303dd05c/util/flowcontrol/throttle.go#L120) of tokenBucketPassiveRateLimiter to check the rate limit. If it exceeds it, [discard the Event](https://github.com/kubernetes/client-go/blob/f6a5a1f1391b9b6ceaaa499bd7cebf508d5e02a6/tools/record/event.go#L218-L220).

## [](#why-did-sawcompletedjob-event-become-unobservable)Why did SawCompletedJob Event become unobservable?

Taking the above into consideration, let's think about why the SawCompleteJob Event became unobservable. In short, it is likely due to be caused by the spam filter.

-   CronJobController issues three Events, SuccessfulCreate, SawCompletedJob, and SuccessfulDelete, per child Job every minute (Strictly speaking, it publishes SuccessfulDelete only when it reaches the HistoryLimit).
-   The Controller uses a spam filter whose the key is solely based on the Source and InvolvedObject (See [getSpamKey function](https://github.com/kubernetes/client-go/blob/2a6c116e406126324eee341e874612a5093bdbb0/tools/record/events_cache.go#L71)). Therefore, these thee types of Events are identified as the same Event.
-   The Controller consumed the first 25 tokens at a rate of three tokens per minute. One token is refilled every five minutes, but around nine minutes, the tokens started running out. After that, A toke was refilled every five minutes, but it was consumed by a (aggregated) SuccessfulCreate Event, so SawCompletedJob and SuccessfulDelete were never published thereafter.

### [](#note-event-types)Note: Event Types

I'll list `SuccessfulCreate`、 `SawCompletedJob`、`SuccessfulDelete` events' involvedObject and source below.

#### [](#successfulcreate-event)SuccessfulCreate Event

```
apiVersion: v1
kind: Event
involvedObject:
  apiVersion: batch/v1
  kind: CronJob
  name: hello
  namespace: default
  resourceVersion: "289520"
  uid: 5f3cfeca-8a83-452a-beb9-7a5f9c1eff63
source:
  component: cronjob-controller
...
reason: SuccessfulCreate
message: Created job hello-28025408
```

Enter fullscreen mode Exit fullscreen mode

#### [](#sawcompletedjob-event)SawCompletedJob Event

```
apiVersion: v1
kind: Event
involvedObject:
  apiVersion: batch/v1
  kind: CronJob
  name: hello
  namespace: default
  resourceVersion: "289020"
  uid: 5f3cfeca-8a83-452a-beb9-7a5f9c1eff63
source:
  component: cronjob-controller
...
reason: SawCompletedJob
message: 'Saw completed job: hello-28025408, status: Complete'
```

Enter fullscreen mode Exit fullscreen mode

#### [](#successfuldelete-event)SuccessfulDelete Event

```
apiVersion: v1
kind: Event
involvedObject:
  apiVersion: batch/v1
  kind: CronJob
  name: hello
  namespace: default
  resourceVersion: "289520"
  uid: 5f3cfeca-8a83-452a-beb9-7a5f9c1eff63
source:
  component: cronjob-controller
...
reason: SuccessfulDelete
message: Deleted job hello-28025408
```

Enter fullscreen mode Exit fullscreen mode