---
title: Learning how an ingress controller works by building one in bash
source: https://community.ops.io/danielepolencic/learning-how-an-ingress-controller-works-by-building-one-in-bash-3fni
clipped: 2023-09-04
published: 2023-01-17
category: network
tags:
  - k8s
  - ingress
  - network
read: true
---

*TL;DR: In this article, you will learn how the Ingress controller works in Kubernetes by building one from scratch in bash.*

Before diving into the code, let's recap how the ingress controller works.

You can think of the ingress controller as a router that forwards traffic to the correct pods.

[![[Raw/Media/Resources/91f5ccac9361c29bb4407b9ddbb78c1a_MD5.png]]](https://community.ops.io/images/x6kM4XvqdvCG_Pn1-L4UNMsgpeONKfV3_BpbyqYhc0k/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vdWFzYWJpL2lt/YWdlL3VwbG9hZC92/MTY3Mzg0Njc3My90/aHJlYWRzL2Jhc2gt/aW5ncmVzcy0yLnBu/Zw)

More specifically, the ingress controller is a reverse proxy that works (mainly) on [L7](https://en.wikipedia.org/wiki/OSI_model) and lets you route traffic based on domain names, paths, etc.

[![[Raw/Media/Resources/53bdf7bd46eec56734f233f5f7a29c9a_MD5.png]]](https://community.ops.io/images/sO9Ucp48n4S__16qTyhRrzvWS1OJkxPNFUpCnNKUvTg/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vdWFzYWJpL2lt/YWdlL3VwbG9hZC92/MTY3Mzg0Njc3My90/aHJlYWRzL2Jhc2gt/aW5ncmVzcy0zLnBu/Zw)

**Kubernetes doesn't come with one by default.**

So you have to install and configure an Ingress controller of choice in your cluster.

But Kubernetes provides an [Ingress manifest (YAML) definition.](https://kubernetes.io/docs/concepts/services-networking/ingress/)

[![[Raw/Media/Resources/906df427aed461bcff4d5458287c88e0_MD5.png]]](https://community.ops.io/images/yl8PhSzSDD3mHei2xODdVq6nuDxD9NUFHeEkm36yhzw/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vdWFzYWJpL2lt/YWdlL3VwbG9hZC92/MTY3Mzg0Njc3My90/aHJlYWRzL2Jhc2gt/aW5ncmVzcy00LnBu/Zw)

The exact YAML definition is expected to work regardless of what Ingress controller you use.

The critical fields in that file are:

1.  The path (`spec.rules[*].http.paths[*].path`).
2.  The backend (`spec.rules[*].http.paths[*].backend.service`).

[![[Raw/Media/Resources/62673a74635218ba25db7aa7c82f7798_MD5.png]]](https://community.ops.io/images/C40-NFKAQ006469TkTWH_q2Sxk8_8-cNxwthVey-YCo/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vdWFzYWJpL2lt/YWdlL3VwbG9hZC92/MTY3Mzg0Njc3My90/aHJlYWRzL2Jhc2gt/aW5ncmVzcy01LnBu/Zw)

The `backend` field describes which service should receive the forwarded traffic.

**But, funny enough, the traffic never reaches it.**

This is because the controller uses endpoints to route the traffic, not the service.

*What is an endpoint?*

[![[Raw/Media/Resources/8ad740aee0c198f6d5b33930c5ba86f7_MD5.png]]](https://community.ops.io/images/ecUw4XHMoZd8sMk-NCixLfxItxYIXQP76nPw8x0haCA/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vdWFzYWJpL2lt/YWdlL3VwbG9hZC92/MTY3Mzg0Njc3My90/aHJlYWRzL2Jhc2gt/aW5ncmVzcy02LnBu/Zw)

When you create a Service, Kubernetes creates a companion [Endpoint object.](https://kubernetes.io/docs/concepts/services-networking/service/#endpoints)

**The Endpoint object contains a list of endpoints (`ip:port` pairs).**

And the IP and ports belong to the Pod.

[![[Raw/Media/Resources/5e46ebca8219bd49a663dd0dc0113a9b_MD5.png]]](https://community.ops.io/images/kTqF1a4mOWWN03UKNsQ5-fiDWZaJPieZHfTJ-mQWLjM/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vdWFzYWJpL2lt/YWdlL3VwbG9hZC92/MTY3Mzg0Njc3My90/aHJlYWRzL2Jhc2gt/aW5ncmVzcy03LnBu/Zw)

*Enough, theory.*

How does this work in practice if you want to build your own controller?

There are two parts:

1.  Retrieving data from Kubernetes.
2.  Reconfiguring the reverse proxy.

Let's start with retrieving the data.

In this step, the **controller has to watch for changes to Ingress manifests and endpoints.**

If an ingress YAML is created, the controller should be configured.

The same happens when the service changes (e.g. a new Pod is added).

[![[Raw/Media/Resources/6799363ec718c0d4d265183bafd43b98_MD5.png]]](https://community.ops.io/images/dnRqp1QlQa9Q5NxJJ6qZC2xzPCKYENpb6GfP24P7rwA/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vdWFzYWJpL2lt/YWdlL3VwbG9hZC92/MTY3Mzg0Njc3My90/aHJlYWRzL2Jhc2gt/aW5ncmVzcy05LnBu/Zw)

In practice, this could be as simple as `kubectl get ingresses` and `kubectl get endpoints <service-name>`.

With this data, you have the following info:

-   The path of the ingress manifest.
-   All the endpoints that should receive traffic.

[![[Raw/Media/Resources/a7d3cc1075195b33c566bd61b82895f4_MD5.png]]](https://community.ops.io/images/aJLZDy-V3uDJOUPiKWcmlUGs1kGsurCggog6_9Wa2bw/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vdWFzYWJpL2lt/YWdlL3VwbG9hZC92/MTY3Mzg0Njc3My90/aHJlYWRzL2Jhc2gt/aW5ncmVzcy0xMC5w/bmc)

With `kubectl get ingresses`, you can get all the ingress manifest and loop through them.

I used `-o jsonpath` to filter the rules and retrieve: the path and the backend service.

[![[Raw/Media/Resources/350636bd2e0e203761569ce9ac939a4f_MD5.png]]](https://community.ops.io/images/UL5xOQi6GhuDbWIL4RzR-uPzeyweJXO5J7yqhTbV0pg/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vdWFzYWJpL2lt/YWdlL3VwbG9hZC92/MTY3Mzg0Njc3My90/aHJlYWRzL2Jhc2gt/aW5ncmVzcy0xMS5w/bmc)

With `kubectl get endpoint <service-name>`, you can retrieve all the endpoints (ip:port pair) for a service.

Even in this case, I used `-o jsonpath` to filter those down and save them in a bash array.

[![[Raw/Media/Resources/6d27e3c350c0ff31b845f0ac9d2b1ead_MD5.png]]](https://community.ops.io/images/WSsoUErH2FQ4P4ybaaAHmcSqi-iXh8LSIfEF2M6_N9I/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vdWFzYWJpL2lt/YWdlL3VwbG9hZC92/MTY3Mzg0Njc3My90/aHJlYWRzL2Jhc2gt/aW5ncmVzcy0xMi5w/bmc)

At this point, you can use the data to reconfigure the ingress controller.

In my experiment, I used Nginx, so I just wrote a template for the `nginx.conf` and hot-reloaded the server.

[![[Raw/Media/Resources/dc442d981e202cbc4e9a80f1a4d7b7b2_MD5.png]]](https://community.ops.io/images/KfMSDEs-oa1HHC4cStOC-h9Ly0j-Emt8bo8tAAIS1Ac/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vdWFzYWJpL2lt/YWdlL3VwbG9hZC92/MTY3Mzg0Njc3My90/aHJlYWRzL2Jhc2gt/aW5ncmVzcy0xMy5w/bmc)

In my script, I didn't bother with detecting changes.

Instead, I decided to recreate the `nginx.conf` in full every second.

[But you can already imagine extending this to more complex scenarios.](https://learnk8s.io/real-time-dashboard)

[![[Raw/Media/Resources/f3a7cc325f37bd1806bfe16d70e383cd_MD5.png]]](https://community.ops.io/images/sHrHi-GBlEmur7QlikVUvjW77m68agm11LYHckhGlXg/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vdWFzYWJpL2lt/YWdlL3VwbG9hZC92/MTY3Mzg0Njc3My90/aHJlYWRzL2Jhc2gt/aW5ncmVzcy0xNC5w/bmc)

The last step was to package the script as a container and set up the proper RBAC rule so that I could consume the API endpoints from the API server.

And here you can find a demo of it working:

[![A demo of a Kubernetes ingress controller written in bash](https://community.ops.io/images/9VjA-jvOTBOkEEpRjA0Dl6QmyOCIbgT35NWMOrkUJqo/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vdWFzYWJpL2lt/YWdlL3VwbG9hZC92/MTY3Mzg0Njc3My90/aHJlYWRzL2Jhc2gt/aW5ncmVzcy1kZW1v/LmdpZg)](https://community.ops.io/images/9VjA-jvOTBOkEEpRjA0Dl6QmyOCIbgT35NWMOrkUJqo/w:880/mb:500000/ar:1/aHR0cHM6Ly9yZXMu/Y2xvdWRpbmFyeS5j/b20vdWFzYWJpL2lt/YWdlL3VwbG9hZC92/MTY3Mzg0Njc3My90/aHJlYWRzL2Jhc2gt/aW5ncmVzcy1kZW1v/LmdpZg)

If you want to play with the code, you can find it here: [https://github.com/learnk8s/bash-ingress](https://github.com/learnk8s/bash-ingress)

I plan to write a longer form article on this; if you are interested, you can sign up for the Learnk8s newsletter here: [https://learnk8s.io/newsletter](https://learnk8s.io/newsletter).

And if you are unsure what ingress controllers are out there, at [Learnk8s](https://learnk8s.io/) we have put together a handy comparison:

[![[Raw/Media/Resources/185c0ddd90cb01379e394298845c7d9c_MD5.png]]](https://docs.google.com/spreadsheets/d/191WWNpjJ2za6-nbG4ZoUMXMpUK8KlCIosvQB0f-oq3k)

And finally, if you've enjoyed this thread, you might also like the Kubernetes workshops that we run at Learnk8s [https://learnk8s.io/training](https://learnk8s.io/training) or this collection of past Twitter threads [https://twitter.com/danielepolencic/status/1298543151901155330](https://twitter.com/danielepolencic/status/1298543151901155330)

Until next time!