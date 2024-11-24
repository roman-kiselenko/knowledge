---
title: Exploring OCI Container Registries by Use Case Pull a Public Image from Kubernetes
source: https://itnext.io/exploring-oci-container-registries-by-use-case-pull-a-public-image-from-kubernetes-3d2d4267201c
clipped: 2023-09-04
published: 
category: k8s
tags:
  - containers
  - oci
read: false
---

![[11e785b4f282f5c05d06c8878c4aa733_MD5.png]]

Let’s begin with one of the most common use cases of a Container Registry:

## Pull A Public Container Image

Here’s a simple kubernetes cluster.

It has access to pull images from the public internet.

![[6d707a21d2d903261963aa015e0d7b05_MD5.png]]

Let’s create a simple pod that runs `hello-world` to completion

➜ kubectl run hello \\  
  --image=hello-world \\  
  --restart=Neverpod/hello created

## Q: Where Does The Image Come From?

To find out, we can examine the logs of the high-level container runtime (sometimes called the container engine). Our OCI Runtime is [Containerd](https://github.com/containerd/containerd).

`containerd.log`

{  
  "msg": "PullImage \\"hello-world:latest\\""  
}  
{  
  "msg": "PullImage using normalized image ref: \\"docker.io/library/hello-world:latest\\""  
}  
{  
  "host": "registry-1.docker.io",  
  "msg": "resolving"  
}  
{  
  "host": "registry-1.docker.io",  
  "msg": "do request",  
  "request.header.accept": "application/vnd.docker.distribution.manifest.v2+json, application/vnd.docker.distribution.manifest.list.v2+json, application/vnd.oci.image.manifest.v1+json, application/vnd.oci.image.index.v1+json, \*/\*",  
  "request.method": "HEAD",  
  "url": "https://registry-1.docker.io/v2/library/hello-world/manifests/latest"  
}

In the Containerd logs, we can see the `hello-world` image is **normalised**.

Specifically, its:

1.  prefixed with the default registry `docker.io`
2.  prefixed with the default repository `library`
3.  suffixed with the default tag `latest`

![[5764ca6921b11310a751fe58a343e4c2_MD5.png]]

Containerd requests the image from `registry-1.docker.io`, better known as DockerHub.

## Q: How Did The Image Get From The Registry To The Container Runtime?

An OCI Image is composed of a **Manifest**, one or more **Filesystem Layers** and an **Image Configuration**.

When Containerd receives a request to run a container from an image, here’s a model of what happens:

![[f5de39a3f909183abaeb35d2104fc51e_MD5.png]]

Did you notice how Containerd precedes each GET request with a check for local presence?

This enables the opportunity for better **efficiency**.

Each OCI Image component is identifiable by its sha256 digest. That digest is derived purely from its content, not by its location.

Containerd applies this knowledge to automatically reduce waste in downloading OCI Image components from the registry. In particular, if a component of the OCI Image exists locally then Containerd skips the download.

We can see an example of the pull sequence using `ctr`, the [CLI client for Containerd](https://github.com/projectatomic/containerd/blob/master/docs/cli.md).

\# ctr image pull \\  
  docker.io/library/hello-world:latestdocker.io/library/hello-world:latest:                                             resolved       |++++++++++++++++++++++++++++++++++++++|   
index-sha256:926fac19d22aa2d60f1a276b66a20eb765fbeea2db5dbdaafeb456ad8ce81598:    done           |++++++++++++++++++++++++++++++++++++++|   
manifest-sha256:7e9b6e7ba2842c91cf49f3e214d04a7a496f8214356f41d81a6e6dcad11f11e3: done           |++++++++++++++++++++++++++++++++++++++|   
layer-sha256:719385e32844401d57ecfd3eacab360bf551a1491c05b85806ed8f1b08d792f6:    done           |++++++++++++++++++++++++++++++++++++++|   
config-sha256:9c7a54a9a43cca047013b82af109fe963fde787f63f9e016fdc3384500c2823d:   done           |++++++++++++++++++++++++++++++++++++++|   
elapsed: 4.8 s                                                                    total:  2.5 Ki (533.0 B/s)                                         
unpacking linux/amd64 sha256:926fac19d22aa2d60f1a276b66a20eb765fbeea2db5dbdaafeb456ad8ce81598...  
done: 7.66556ms

The Container Runtime first downloads the Image Index, then the Image Manifest. The Image Manifest contains digests of the Layers and the Configuration. The layers are blobs (binary large objects). The Index, Manifest and Config are JSON documents.

The Container Runtime can detect changes in a Manifest, Layer or Configuration by computing the content digest (`sha256sum [FILE]`) and comparing it to the identifier digest.

:bulb: If the digests match, there are no changes. Its the same content. It doesn’t matter where you download it from or where you store it! \*.

This design choice is called [Content Addressable Storage](https://en.wikipedia.org/wiki/Content-addressable_storage).

-   Content Addressable storage can enable better distribution and storage efficiency in Registry and Runtime.

> \* Instead, what matters is a way to trust the creator of the image. If you can trust the digest of the initial Image Index or Image Manifest, then you can trust the rest of the content! In practice, this is typically achieved by signing images.

## More Clusters, More Image Pulls

Imagine an organisation has 6 application teams.

Each team has their own cluster so they can operate independently on their own cadence.

Again, each cluster has access to pull images from the public internet.

![[13d81d49ee54eeb7c747b33b5cd80579_MD5.png]]

Actually, each team wants to run a job that tests a matrix of 3 image versions.

Additionally, it must run to completion exactly 6 times and must complete quickly.

Finally, these images have mutable tags and its important to test with the freshest.

Here’s the job specification:

`hello-job.yaml`

apiVersion: batch/v1  
kind: Job  
metadata:  
  name: hello  
spec:  
  completions: 6  
  parallelism: 6  
  activeDeadlineSeconds: 600  
  template:  
    metadata:  
      labels:  
        app: hello  
    spec:  
      containers:  
      - image: hello-world:linux  
        name: hello-linux  
        imagePullPolicy: Always  
      - image: hello-world:nanoserver-ltsc2022  
        name: hello-nanoserver  
        imagePullPolicy: Always  
      - image: hello-world:nanoserver-1809  
        name: hello  
        imagePullPolicy: Always  
      restartPolicy: Never  
      affinity:  
        podAntiAffinity:  
          requiredDuringSchedulingIgnoredDuringExecution:  
          - labelSelector:  
              matchExpressions:  
              - key: app  
                operator: In  
                values:  
                - hello  
            topologyKey: kubernetes.io/hostname

Monday, first thing, the teams deploy the jobs. What happens next?

## Problem: `Errimagepull`, Hit The Limit

Two minutes later, we’re seeing ErrImagePull errors…

$ kubectl get events90s         Warning   Failed                    pod/hello-zhkp7                 
Failed to pull image "hello-world:nanoserver":   
rpc error: code = Unknown desc = failed to pull and unpack image "docker.io/library/hello-world:nanoserver":   
failed to copy: httpReadSeeker: failed open:   
unexpected status code https://registry-1.docker.io/v2/library/hello-world/manifests/sha256:3cabdfb783cd2710153b3824ba5d94c8ebecc0bc48251e2e823f82a15dec660f:   
429 Too Many Requests - Server message:   
toomanyrequests: You have reached your pull rate limit. You may increase the limit by authenticating and upgrading: [https://www.docker.com/increase-rate-limit](https://www.docker.com/increase-rate-limit)

If we examine the events, we got a `429 Too Many Requests` response from DockerHub.

DockerHub [limits the number of container image pulls](https://docs.docker.com/docker-hub/download-rate-limit/) based on the account type of the user pulling the image. DockerHub identifies anonymous (i.e. unauthenticated) users by their source IP address.

To summarise:

|Account Type|Limit|  
|---|---|  
|anonymous users| 100 pulls per 6 hours per IP address.|  
|authenticated users| 200 pulls per 6 hour period.|  
|Users with a paid Docker subscription| 5000 pulls per day.|

We can visualise the remaining requests with the handy [Docker Hub Rate Limit Exporter for Prometheus](https://gitlab.com/gitlab-de/unmaintained/docker-hub-limit-exporter/?_gl=1*y0hdof*_ga*MTY2MTE5MTAxOC4xNjcxMzQ4ODM1*_ga_ENFH3X7M5Y*MTY4ODgyNTU4NS42LjAuMTY4ODgyNTYwNC4wLjAuMA..).

![[0b1e6b83459407dd904bede424693ef0_MD5.png]]

Yup, Remaining Requests is 0! We hit the DockerHub rate limit!

How did that happen?

1.  we have 6 clusters
2.  each cluster has 6 worker nodes
3.  each team deployed a job that ran 6 `hello` pods to completion, in parallel - fancy stuff!
4.  for freshness, each hello-world pod `Always` attempts to pull 3 versions of the `hello-world` image.

That’s **6 clusters \* 6 pods \* 3 containers = 108 image pulls**

![[09d80abcd3c5da7999060f77e7dc3f98_MD5.png]]

## Q: What *is* An Image Pull Request?

:bulb: An Image **Pull Request** is [defined by Docker Inc.](https://docs.docker.com/docker-hub/download-rate-limit/#definition-of-limits) as:

-   One or two `GET` requests on registry manifest URLs (`/v2/*/manifests/*`).
-   There’ll be two requests if there’s an Image Index.
-   `HEAD` requests aren’t counted.

## Q: What About The 6 Worker Nodes? Why Is That Significant?

Kubernetes spreads the 6 pods across the worker nodes because of the `podAntiAffinity` rule in the pod spec.

The `hello-world` OCI Images don't exist on a worker node by default.

\# ctr images list --quiet | grep hello-world

Before it can create and start a container, each worker must pull the image from DockerHub.

Its significant because Dockerhub receives many requests in a short time frame.

But each worker has its own IP address on the network.

Why then does Dockerhub identify them as one single unauthenticated puller?

## Q: Why Are Workers Sharing The Dockerhub Limit?

Let’s examine the response from DockerHub.

Again, from the Containerd logs, the response looks like this:

`containerd.log`

{  
  "host": "registry-1.docker.io",  
  "msg": "fetch response received",  
  "response.header.content-length": "2561",  
  "response.header.content-type": "application/vnd.docker.distribution.manifest.list.v2+json",  
  "response.header.docker-content-digest": "sha256:fc6cf906cbfa013e80938cdf0bb199fbdbb86d6e3e013783e5a766f50f5dbce0",  
  "response.header.docker-distribution-api-version": "registry/2.0",  
  "response.header.docker-ratelimit-source": "58.185.1.1",  
  "response.header.etag": "\\"sha256:fc6cf906cbfa013e80938cdf0bb199fbdbb86d6e3e013783e5a766f50f5dbce0\\"",  
  "response.header.ratelimit-limit": "100;w=21600",  
  "response.header.ratelimit-remaining": "99;w=21600",  
  "response.header.strict-transport-security": "max-age=31536000",  
  "response.status": "200 OK",  
  "url": "https://registry-1.docker.io/v2/library/hello-world/manifests/latest"  
}

Notice the `response.header.docker-ratelimit-source`. Its `58.185.1.1`.

Thats the public IP address of the network’s internet gateway. Its the source address that DockerHub sees.

This happens if Source Network Address Translation (SNAT) is configured for outbound internet requests.

![[c8b36f35ce3960fcb01f0c4fae6abebf_MD5.png]]

The result is each request has the same IP address no matter which cluster originates the request.

If you’re in an organisation with many clusters, and those clusters pull images from Dockerhub through a SNAT gateway, in the same way, you can hit the limit very quickly!

## Q: How Might We Work Around The Pull Limit?

There are a couple of alternatives to DockerHub here:

## 1\. Pull From A Different Public Registry

> If you’re using AWS EKS, you can pull the majority of popular docker images from ECR Public Registry.
> 
> For example `docker pull public.ecr.aws/docker/library/hello-world:latest`
> 
> On AWS, its logically closer to your infrastructure and you wont encounter any rate limiting.

## 2\. Operate Your Own Private OCI Registry

> If you already have a central binary repository in your org like a managed Artifactory, Nexus or the Harbor, you’re likely already doing this.
> 
> For example `docker pull containers.your.org/library/hello-world:latest`
> 
> This solution becomes increasingly compelling as your container consumption grows.

We’re gonna choose option #2, but we wont use a vendor product because we wanna learn with the simplest components that meet the OCI specifications!

## Create A Private Proxy Cache OCI Registry For Dockerhub

The simplest OCI Registry is a container running the `registry:2` image from [distribution/distribution](https://github.com/distribution/distribution/releases) :

➜ k3d registry create docker-io-mirror \\  
\--image docker.io/library/registry:2 \\  
\--port 0.0.0.0:5005 \\  
\--proxy-remote-url https://registry-1.docker.io \\  
\--volume /tmp/reg:/var/lib/registry \\  
\--volume $(pwd)/registry-config.yml:/etc/docker/registry/config.yml \\  
\--no-helpINFO\[0000\] Creating node 'k3d-docker-io-mirror'           
INFO\[0000\] Successfully created registry 'k3d-docker-io-mirror'   
INFO\[0000\] Starting Node 'k3d-docker-io-mirror'           
INFO\[0000\] Successfully created registry 'k3d-docker-io-mirror'

Initially, the registry is empty:

\# wget k3d-docker-io-mirror:5000/v2/\_catalog -qO-{  
  "repositories": \[\]  
}

Next, let’s pull the image again from one of our kubernetes worker using `ctr`.

This time we’ll pull it from our brand new private OCI Registry.

\# ctr image pull \\  
  --plain-http \\  
  k3d-docker-io-mirror:5000/library/hello-world:linux

Notice we need to specify both the **registry** and **repository** prefix explicitly in the image identifier.

![[85ca764cc1efa450e4a64373e67321ea_MD5.png]]

Indeed, the normalisation to docker.io and library are historical hangovers from the era when Docker’s official images on Dockerhub were the only game in town!

And here’s the output of `ctr image pull`:

k3d-docker-io-mirror:5000/library/hello-world:linux:                              resolved       |++++++++++++++++++++++++++++++++++++++|   
index-sha256:726023f73a8fc5103fa6776d48090539042cb822531c6b751b1f6dd18cb5705d:    done           |++++++++++++++++++++++++++++++++++++++|   
manifest-sha256:7e9b6e7ba2842c91cf49f3e214d04a7a496f8214356f41d81a6e6dcad11f11e3: done           |++++++++++++++++++++++++++++++++++++++|   
config-sha256:9c7a54a9a43cca047013b82af109fe963fde787f63f9e016fdc3384500c2823d:   done           |++++++++++++++++++++++++++++++++++++++|   
layer-sha256:719385e32844401d57ecfd3eacab360bf551a1491c05b85806ed8f1b08d792f6:    done           |++++++++++++++++++++++++++++++++++++++|   
elapsed: 5.3 s                                                                    total:  4.9 Ki (952.0 B/s)                                         
unpacking linux/amd64 sha256:726023f73a8fc5103fa6776d48090539042cb822531c6b751b1f6dd18cb5705d...  
done: 40.463514ms

Notice also that the first pull from the empty registry took **5.3 seconds**.

Let’s see if `hello-world` is there in the k3d-docker-io-mirror...

➜ wget k3d-docker-io-mirror:5000/v2/\_catalog -qO-{  
  "repositories": \[  
    "library/hello-world"  
  \]  
}

It is!

Our private registry cached the image.

## Repeat The Experiment: 6 Clusters x 6 Pods x 3 Containers = 108 Image Pulls

Now we have a private registry working, let’s turn our attention back to our multi-cluster test scenario.

We’re gonna repeat the experiment.

Let’s update the images in the pod spec to pull from our private registry.

Here’s the updated job specification:

`hello-job-private-reg.yaml`

apiVersion: batch/v1  
kind: Job  
metadata:  
  name: hello  
spec:  
  completions: 6  
  parallelism: 6  
  activeDeadlineSeconds: 600  
  template:  
    metadata:  
      labels:  
        app: hello  
    spec:  
      containers:  
      - image: k3d-docker-io-mirror:5000/library/hello-world:linux  
        name: hello-linux  
        imagePullPolicy: Always  
      - image: k3d-docker-io-mirror:5000/library/hello-world:nanoserver-ltsc2022  
        name: hello-nanoserver  
        imagePullPolicy: Always  
      restartPolicy: Never  
      affinity:  
        podAntiAffinity:  
          requiredDuringSchedulingIgnoredDuringExecution:  
          - labelSelector:  
              matchExpressions:  
              - key: app  
                operator: In  
                values:  
                - hello  
            topologyKey: kubernetes.io/hostname

After re-applying the `hello` from private registry job definition, the container runtime should pull the image from our private OCI registry.

![[7355a92e36b6be7e29bcabea5442f2ed_MD5.png]]

But is there any difference from the first time?

kubectl describe pod hello-vksvdEvents:  
  Reason                       Message  
  ------                       -------  
  Scheduled                    Successfully assigned default/hello-vksvd to k3d-cluster-5-agent-2  
  Pulling                      Pulling image "k3d-docker-io-mirror:5000/library/hello-world:linux"  
  Pulled                       Successfully pulled image "k3d-docker-io-mirror:5000/library/hello-world:linux" in 1.316861913s (1.316868009s including waiting)  
  Created                      Created container hello-linux  
  Started                      Started container hello-linux

Yup! This time its 1.3 seconds! What happened?

Here’s what happened exactly:

![[b08dafa943b655f76920d36d410e847f_MD5.png]]

1.  Fetch the **OCI Image Manifest** digest. **Containerd** makes a HEAD request to the registry mirror at `/v2/library/hello-world/manifests/linux?ns=docker.io`.
2.  **docker-io-mirror** forwards the HEAD request to Dockerhub to check if the digest for the tag `linux` has changed.
3.  **Dockerhub** responds with the sha256 digest of the Image Manifest.
4.  **docker-io-mirror** then responds with the sha256 digest of the Image Manifest.

Since `hello-world:linux` already exists in the Private Registry, it made only one `HEAD` request to Dockerhub to fetch the identity of the Manifest - its sha256 digest.

The manifest’s sha256 digest is all that’s needed to determine that nothing had changed. All of the required layers and configuration are already present on docker-io-mirror.

The result is faster pulls. There are fewer requests to Dockerhub and we get lower latency on requests for manifest and layer downloads from the local Registry Mirror.

How about the `nanoserver-ltsc2022` image?

![[145d32d3f2ceb88a293865babd47e222_MD5.png]]

1.  Containerd receives the Image Index in reponse to the Manifest Download.
2.  Containerd determines there’s no matching **Manifest** for platform (linux, amd64) in the **Image Index**.

It’s a Windows image so Containerd won’t pull it on Linux and it won’t pull it on Mac. However, it will still make a GET request to the registry for the Image Index and that will counted as a pull by DockerHub.

## Q: What Happened To Our Docker Pull Requests Limit Now?

`--image-pull-policy=Always` insists Containerd to pull from the registry rather than use the image stored locally on the worker.

Since each container in our pod spec has `imagePullPolicy: Always`, we can expect Containerd to pull from our private registry on each container create operation.

![[3a42e406546ac58fff87ca4ebf5ac71a_MD5.png]]

This time, it used only 2 pull requests!

1.  One to GET the Image Index and Manifest for `hello-world:linux`. Containerd made these GET requests when we used `ctr` to pre-pull the image.

-   Remember: 1 Dockerhub pull request is one or two `GET` requests on registry manifest URLs (`/v2/*/manifests/*`).

1.  One to GET the Image Index for `hello-world:nanoserver-ltsc2022`. Containerd determines it's platform (architecture and os) are not supported for this image.

Thank you for reading this article right to the end. If you enjoyed it and if you think others can benefit, please like and share!

If you’d like to learn from a hands-on-keyboard tutorial for this chapter, let me know on [LinkedIn](https://www.linkedin.com/in/doughellinger/).

If you foresee a problem, have an alternative solution, I’d appreciate your feedback. Again, reach me on [LinkedIn](https://www.linkedin.com/in/doughellinger/).

Special thank you to [Dan Polencic](https://www.linkedin.com/in/danielepolencic/). Appreciate the reviews and all your feedback!

Look out for the next chapter… Exploring OCI Container Registries: Chapter 2: Pull a Public Helm Chart