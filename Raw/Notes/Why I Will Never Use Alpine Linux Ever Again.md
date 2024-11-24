---
title: Why I Will Never Use Alpine Linux Ever Again
source: https://martinheinz.dev/blog/92
clipped: 2023-09-04
published: 2023-03-06
category: development
tags:
  - containers
read: true
---

Nowadays, Alpine Linux is one of the most popular options for container base images. Many people (maybe including you) use it for anything and everything. Some people use it because of its small size, some because of habit and some, just because they copy-pasted a `Dockefile` from some tutorial. Yet, there are plenty of reasons why you **should not** use Alpine for your container images, some of which can cause you great amount of grief...

## The Source of All The Grief

To understand what makes Alpine a bad choice in some situations, we first need to talk about `musl`. `musl` is an implementation of C standard library. It is more lightweight, faster and simpler than `glibc` used by other Linux distros, such as Ubuntu. Both of these implementations are interchangeable for the most part, that's why in most cases you can switch from e.g., Ubuntu to Alpine and never notice any difference.

However, the little differences can cause all the grief. Some of it stems from how `musl` (and therefore also Alpine) handles DNS (it's always DNS), more specifically, `musl` (by design) doesn't support [DNS-over-TCP](https://serverfault.com/a/404843). Usually, you would not notice this difference, because most of the time a single UDP packet (512 bytes) is enough to resolve hostnames... until it isn't enough and your application (running on Kubernetes) that previously worked completely fine for months suddenly starts throwing *"Unknown Host"* exceptions for one particular (very critical) hostname. The worst part is that this can manifest randomly, anytime when some external network change causes the resolution of some particular domain to require more than the 512 bytes available in single UDP packet.

***By using Alpine, you're getting "free" chaos engineering for you cluster.***

For more details about this particular problem check out this great write-up: [Does Alpine resolve DNS properly?](https://purplecarrot.co.uk/post/2021-09-04-does_alpine-resolve_dns_properly/)

If you run tens or even hundreds of microservices/applications all based on Alpine, and they all suddenly stop working and the only fix is to switch to different Linux distro, which requires rebuilding all the applications and redeploying them, then you can be faced with extremely disruptive, multi-day outage.

Finally, this DNS issue does not manifest in Docker container. It can only happen in Kubernetes, so if you test locally, everything will work fine, and you will only find out about unfixable issue when you deploy the application to a cluster. Also, [Kubernetes docs](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/#known-issues/) claim that DNS issues are relevant only for *"Alpine version 3.3 or earlier"*, but I encountered the above issue on Alpine 3.16, so go figure.

As a little bonus many popular tools also use Alpine as a base image, such `nicolaka/netshoot` or `giantswarm/tiny-tools`, with the former being specifically targeted at network troubleshooting. Good luck troubleshooting your networking when your troubleshooting tool is also broken.

## Beyond DNS Issues

While DNS is the most common issue with `musl`, there are more reasons to reconsider using it. Any programming language or its library that relies on C standard library is impacted by any and all differences between `musl` and `glibc`.

For Python, for example, many popular libraries such as NumPy, or [Cryptography](https://stackoverflow.com/questions/35736598/cannot-pip-install-cryptography-in-docker-alpine-linux-3-3-with-openssl-1-0-2g) rely on C code for optimizations. Luckily, at least for some of the libraries like Numpy, chances are, you will find Alpine-based compiled package and relevant dependencies. For the less popular ones however, you might have to compile them yourself and is it really worth the hassle? In my opinion... nope. Additionally, even if you manage to build an image that includes for example `numpy`, its size will be ~400MB, at which point using Alpine for its small size doesn't really help much.

Also, the build time for such an image will be atrocious, you can try for yourself, following `Dockerfile` takes almost 10 minutes to build:

```
FROM python:3.11-alpine
RUN apk --update add gcc build-base
RUN pip install --no-cache-dir numpy
```

Obviously, similar issues can happen also in other languages. For example Node.js uses addons, which are written in C++ and compiled with `node-gyp`, these will depend on C libraries and therefore on `glibc`.

Another example is Golang whose standard library - or more specifically `net/http` or `os/user` modules - depend on C libraries and therefore on `glibc`. Even if you don't use those particular modules, if your application requires `CGO_ENABLED=1`, you will obviously run into issue with Alpine.

## What to Use Instead?

If the above issues prompted you to reconsider using Alpine, then you might be wondering what to use instead. There are plenty of options, all of them have some pros/cons/tradeoffs.

The biggest appeal of Alpine is its small size, so if you really care about that, then [Wolfi](https://github.com/wolfi-dev/) (e.g. `cgr.dev/chainguard/wolfi-base` is just 12MB) or [Distroless](https://github.com/GoogleContainerTools/distroless) are good choices.

If you're looking for general purpose base image with reasonable size that's not based on `musl`, then you might consider using *UBI (Universal Base Image)* made by Red Hat, which has only 26.7MB in its *"micro"* version (`registry.access.redhat.com/ubi8-micro`) which is also fairly close to Alpine.

Another reason to choose Alpine is because of security. This also relates to its small size, because the small size generally means fewer packages and therefore also fewer vulnerabilities. The above-mentioned Wolfi images are especially good choice in this regard.

Let's be real though, chances are that saving a couple megabytes of space thanks to Alpine being small won't matter unless you're pulling the image thousands of times (which you probably shouldn't be doing anyway), so using Ubuntu or Debian-based base image wouldn't be a bad choice either.

## Conclusion

While there's nothing wrong with using Alpine and it *can* be great a base container image OS, I - personally - will probably never trust it again, or anything that uses `musl` for that matter, due to the DNS issues described earlier.

Point of this article is not to crap on Alpine, rather it's meant to serve as warning. While Alpine might seem like a good choice - considering above listed issues - using it is at very least risky, possible reckless depending on where you plan to use it. It however ticks so many boxes and has a lot of pros, so if you're not worried about or are not impacted by the issues described in this article, you should probably keep on using it.

With all that said, takeaway here should be - do a reasonable amount of research before committing to using *anything* (whether it's container OS, framework, library) - and it being popular and well regarded doesn't necessarily mean it's automatically a good choice.