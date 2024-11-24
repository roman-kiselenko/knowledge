---
title: How we reduced our docker build times by 40%
source: https://medium.com/datamindedbe/how-we-reduced-our-docker-build-times-by-40-afea7b7f5fe7
clipped: 2023-11-15
published: 
category: development
tags:
  - docker
  - containers
read: true
---

Similar to many companies, we build docker images for all components that are used in our product. Over time, a couple of these images became bigger and bigger and also our CI builds were taking longer and longer. My goal is that CI builds don’t take longer than 5 minutes. The idea comes from the fact that it is the ideal length for a coffee break. When builds take longer than that, it slows down the developer productivity.

The reason for the loss in productivity is caused by:

-   developers need to wait for the build to complete and thus waste time
-   developers start on something new and come back to it at a later time. This requires more context switching which often also leads to inefficiencies.

![[Raw/Media/Resources/55c42895e7e4931c72e300a730beffa5_MD5.png]]

In this blogpost, I want to illustrate 2 small changes that we applied and that resulted in a drastic improvement of our build times. Before focussing on these improvements, make sure you already follow the [best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/) like:

-   minimize the number of layers
-   use multistage builds
-   use a minimal base image
-   …

Lets start by explaining Buildkit and Buildx as both terms are often used interchangeably but they are not the same. Before writing this post, I did not fully understand the difference between both either.

## **Builkit**

[Buildkit](https://docs.docker.com/build/buildkit/) is the improved backend to replace the legacy Docker builder. It is packaged with Docker as of 2018 and became the default builder as of docker engine 23.0.

It provides a lot of interesting features:

-   improved caching capabilities
-   parallelize building distinct layers
-   lazy pulling the base image (≥ Buildkit 0.9)

When using Buildkit, you quickly notice that the output of the *docker build* command looks cleaner and more structured.

A typical way to use Buildkit with a docker version older than 23.0 is to set the Buildkit argument as follows:

DOCKER\_BUILDKIT=1 docker build --platform linux/amd64 . -t someImage:someVersion  
DOCKER\_BUILDKIT=1 docker push someImage:someVersion

## **Buildx**

[Buildx](https://docs.docker.com/engine/reference/commandline/buildx/) is a plugin for Docker that **enables you to use the full potential of Buildkit** in Docker. It was created because Buildkit supports many new configuration options, that cannot all be integrated into the *docker build* command in a backwards compatible way.

On top from building images, Buildx supports managing multiple builders. This can be useful in CI to define scoped environments, with distinct configuration, as they do not modify the shared Docker daemon.

You can get started with Buildx as follows:

docker buildx create --bootstrap --name builder  
docker buildx use builder

A first way to speed up your builds is to cache your image in a remote registry. This way you can benefit from the build cache even when your build is performed on a different machine, as is typically the case in CI. As a workaround, many people pulled the latest version of the image before building a new image version. The benefit is that you can cache the layers that did not change at the cost of pulling the full image initially. Pulling the full image can take a while but there is also no guarantee that layers can be reused. To illustrate, we used the following commands:

docker pull someImage:latest || true  
docker build --platform linux/amd64 . \\  
\-t someImage:someVersion \\  
\-f Dockerfile \\  
\--cache-from someImage:latest

With Buildx, you can store the cache information in a remote location (e.g. container registry, blob storage, …). The builder checks whether a given layer already exists and if this is the case it will reuse it instead of creating it again. This can even be done without pulling the layer locally. To benefit from this mechanism, we reworked the previous commands to:

docker buildx build --platform linux/amd64 . \\  
\-t someImage:someVersion - push \\  
\--cache-to type\=registry,ref=someCachedImage:someVersion,mode=max  
\--cache-from type\=registry,ref=someCachedImage:someVersion

*Mode “max”* means that we will store build information for every layer, even layers that are not used in the resulting image (e.g. when using multi stage builds). By default mode *“min”* is used*,* which only stores build information about the layers that exist in the final image.

A special case of caching is to store the cache data “*inline”*, which means it will be cached together with the image. This option is also supported when using Buildkit without Buildx. It is the easiest to start with but is more tricky when using multi-stage builds and it does not provide a clear separation between the output of the artifacts and the cache. The commands to store the cache data “*inline”* looks as follows:

docker buildx build - platform linux/amd64 . \\  
\-t someImage:someVersion --push \\  
\--cache-to type\=inline,mode=max \\  
\--cache-from someImage:somePreviousVersion

Docker introduced new version of it’s syntax for writing dockerfiles, namely: *#syntax=docker/dockerfile:1.4.* It supports an extra link option for *COPY* and *ADD* commands.

Previously, when you use the *COPY* or *ADD* command the builder created a new snapshot, which merges the new files with the already existing file system. The consequence is that the parent layers all need to exist before this operation can be performed as otherwise the destination directory might not exist yet. In the end your image (the result of the build command) will consist of tarballs per layer, which contain the diff between the respective snapshots.

FROM baseImage:version  
COPY binary /opt/

When using the link option, new files are put in their own snapshot without depending on the previous layers. The linked files are stored in their own tarball and the different tarballs are linked together, without depending on the existing file system as is illustrated by the following image.

![[Raw/Media/Resources/b4dbf7be9b7a5d0a4d0e8aede8a54b14_MD5.png]]

[https://www.docker.com/blog/image-rebase-and-improved-remote-cache-support-in-new-buildkit/](https://www.docker.com/blog/image-rebase-and-improved-remote-cache-support-in-new-buildkit/)

  
FROM baseImage:version  
COPY \[--chown\=<user>:<group>\] \[--chmod\=<perms>\] --link binary /opt/

The major advantage is that the files are not dependent on previous layers anymore. The layer can be reused as long as the files did not change, even if the parent layers changed.

Additionally, this can also improve the speed of your builds as multiple layers copying data can now be executed in parallel.

This blogpost describes some new insights, that we gained after optimising our CI pipelines. I discuss 2 small changes that resulted in a **40 percent reduction** of our overall **docker build times**:

-   Storing the **build cache information remotely**
-   Using the **link option** when **adding,** **copying files** into your docker image