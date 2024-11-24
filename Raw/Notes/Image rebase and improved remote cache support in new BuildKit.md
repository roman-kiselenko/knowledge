---
title: Image rebase and improved remote cache support in new BuildKit
source: https://www.docker.com/blog/image-rebase-and-improved-remote-cache-support-in-new-buildkit/
clipped: 2023-11-02
published: 
category: development
tags:
  - docker
read: false
---

We’ve just shipped new versions of the [BuildKit](https://github.com/moby/buildkit/releases/tag/v0.10.0) builder engine, [Dockerfile 1.4](https://github.com/moby/buildkit/releases/tag/dockerfile%2F1.4.0) frontend, and [Docker Buildx CLI](https://github.com/docker/buildx/releases/tag/v0.8.0). Each of these comes with many new features. In this blog post, I’ll show one of them, a new copy mode in Dockerfiles, and explain why you should start to use it on your Dockerfiles.

With the Dockerfile 1.4 release, the `COPY` and `ADD` commands for copying files from the build context or from another stage now accept [a new flag \`–link\`](https://github.com/moby/buildkit/blob/dockerfile/1.4.0/frontend/dockerfile/docs/syntax.md#linked-copies-copy---link-add---link). Using this flag enables much better cache semantics as well as the ability to perform a fast 2nd-day rebase of your builds on top of new base images without rebuilding them.

In order to use this flag, you will need to add a line containing `# syntax=docker/dockerfile:1.4` to the top of your Dockerfile. This makes sure that the proper frontend image with support for this flag is loaded. In order to get the correct cache semantics for the flag, BuildKit v0.10 needs to be used as well.

```docker
# syntax=docker/dockerfile:1.4
FROM ...
COPY --link foo bar
```

```shell
docker buildx create --use --name mybuilder
docker buildx build .
```

Before we get into the details of what this new flag does, let’s go over how the Dockerfile commands work at the moment.

Docker images consist of layers that are tarballs in the registry that make up the container filesystem. When you pull an image, these tarballs get extracted on top of each other. The implementation of how this extraction happens and how files actually get stored on the disk depends on the underlying snapshotter type. If you use the overlay snapshotter, your filesystem can create a special mount that combines multiple directories into one. For other snapshotters the process usually involves making (shallow) copies of files.

Every `RUN`, `COPY` or `ADD` command in Dockerfile also creates a new snapshot that is added on top of previously created contents. Once the build is ready and you want to export an image as a build result, we will run a “differ” component that compares all the snapshots and creates new tarballs containing the new files that were added in each snapshot.

An important concept to understand here is that in order for a new layer to be created, the previous layers(also called parent layers) already need to be created before and exist on disk. Whenever you used `COPY` command to move some files to a directory, all the previous commands on the same stage needed to be completed before. Without it, you wouldn’t have the destination directory where the files would be copied to.

This limitation changes now with the new `--link` flag that has been added to `COPY` and `ADD` commands. When this flag is present, the `COPY` command works in a different mode where files are instead copied to a completely new snapshot. Then this new snapshot is turned into a new layer tarball on its own, and that tarball is linked into the chain of previous tarball layers. This linking action is usually just a metadata change where a new item is added to the layers array without the need to access or move any files. As shown in the next examples, it can even happen remotely with the layers existing in the remote registry without ever needing to pull or push them.

 ![[Raw/Media/Resources/533341f5fb141eb93352891a7983e8c9_MD5.png]]

To summarize:

-   `COPY --link=false` (previous method and default): Files are copied on top of the result of the previous command Layers are created later by comparing snapshots on disk
-   `COPY --link=true`: Files are copied to a new location and turned into an independent layer Layer identifier is added on top of previous layers

By removing the dependency from the destination directory, we don’t need to wait for previous commands to finish before completing the `COPY` command. We also do not need to invalidate our build cache for the current command when previous commands on the same Dockerfile stage change.

Let’s look at some example use-cases that this enables.

### Example: Rebasing an existing image

The previous release of BuildKit v0.9 introduced another new feature: lazy image pulling. What this feature means is that whenever BuildKit needs to access a remote image/cache, it will delay the pulling of its layers until there is a task that actually needs to read files from them. For example, when a layer is just used in another image this pulling is not needed and BuildKit can just create a new image referencing the previous layer by its immutable digest.

```docker
FROM ubuntu
ENV MYCONFIG=foo
VOLUME /data
```

For example, if you build this Dockerfile with `docker buildx build -t myuser/myubuntu --push .` on a clean system without cache, you will notice that the whole build only takes a couple of seconds before your new image is ready in your repository. This is because the layers of the ubuntu image are never pulled to your local machine and never pushed to hub repository. Instead, BuildKit creates a new image config and manifest containing the Ubuntu layer digests and pushes only them. The layers are linked directly from the Ubuntu repository using the cross-repo mount feature of the registry. This pattern can also be used with a remote cache source where your build would only need to validate that remote cache is still up-to-date and not actually pull down any layers.

This method works well for metadata commands like `ENV` and `VOLUME` that only modify the image config. If you used a command that created new layers like `COPY` or `RUN`, the base image still needed to be pulled first because local files were needed in order to run these commands.

`COPY --link` removes this requirement. Let’s look at a common multi-stage build Dockerfile that has been updated to use `COPY --link`:

```docker
#syntax=docker/dockerfile:1.4
FROM golang AS build
....
RUN go build -o /myapp .

FROM alpine:3.14
COPY --from=build --link /out/myapp /bin
ENTRYPOINT \["/bin/myapp"\]
```

When you build this file with BuildKit v0.10, the first thing you will notice is that your build completes without ever pulling the Alpine image. This is because copying `myapp` to the `/bin/` directory does not depend on Alpine files anymore. If you push this image to another Docker Hub repository Alpine layers are linked directly. Only if you export the image in some other way, for example into a local OCI tarball with `--output type=oci` will the layers be actually pulled.

Now when we have built and pushed this image for the first time, we can look at what happens when we need to update this image in the future. Either in the case a new Alpine 3.14 image with security fixes comes out or when we want to update to 3.15.

To avoid rebuilding everything again we can store remote cache from our earlier build. BuildKit supports many cache backend but the easiest, in this case, is to use “inline cache” that just embeds the build cache information into the image config.

To enable inline cache we either run:

`docker buildx build --cache-to type=inline --push -t mysuser/myapp .`

or

`docker buildx build --build-arg BUILDKIT_INLINE_CACHE=1 --push -t mysuser/myapp .`

Now we can use the image itself as a cache source when doing subsequent builds. For example, let’s see what happens when we update our previous Dockerfile to use Alpine 3.15 instead and build using the previous cache.

```docker
FROM golang AS build
....
FROM alpine:3.15
COPY --from=build --link /out/myapp /bin/
ENTRYPOINT \["/bin/myapp"\]
```

`docker buildx build --cache-from myuser/myapp -t myuser/myapp --push .`

Similarily to our initial build, we will see that `alpine:3.15` is not actually pulled to the local machine, and instead, the layer blobs were directly moved inside the registry. What might be more interesting is that the `golang` image was not pulled as well. This is because we can verify that the `myapp` binary has not changed, and therefore the second layer in our image has not changed as well, and we can just rebase it on top of the new alpine image. This all happens completely remotely without any local layers.

Note that without `--link` this was not possible before as the `COPY` operation depended on `/bin` directory from the base image and its cache was not valid anymore because the base image changed, resulting in pulling both Alpine and Golang image and recompilation of `myapp` binary.

### Example: Better remote cache support

As another example, let’s look at how the cache is handled if you have multiple `COPY` commands.

```docker
#syntax=docker/dockerfile:1.4
FROM golang AS build
....
RUN go build -o /myapp .

FROM ubuntu AS config
...
RUN generate -o /myapp.config

FROM alpine:3.14
COPY --from=config --link /myapp.config /etc/
COPY --from=build --link /myapp /bin/
ENTRYPOINT \["/bin/myapp"\]
```

In this file, we have added a second copy that adds a generated config file from another build stage. It is a very common pattern to use multiple stages for dependencies and then copy them all together in a final stage. This is how you get the best parallelization and cache reuse for your builds.

Let’s say we build and push this Dockerfile with inline cache as before:

`docker buildx build --cache-to type=inline -t myuser/myapp2 --push .`

Now let’s consider what happens when we need to do a rebuild using our previous inline cache and our config file generation has changed. The stage with our config generation needs to run again, but what happens to the last stage?

Without using `--link`, if the file `myapp.config` changed it would mean that Alpine image was pulled and extracted, `myapp.config` copied over that snapshot, and because that changed the dependencies for the `COPY` of `myapp` it would need to be recompiled and copied again as well. Note that the possibility of cache reuse here depended on the order of commands, the cache could be used until the last `COPY` command that matched cache, and all commands after that would need to run again. If cache for `myapp` would have been invalidated, we still would have gotten cache for `myapp.config` because that file was copied before, but not vice versa.

By adding `--link`, the cache reuse is now much better. All the `COPY` commands are now independent and none of them depend on the base image. After the new config is generated, it is directly converted into a new layer. Then this layer is replaced inside the previous image. The bottom layers for the base image and the top layer containing `myapp` are left as is – they never need to be pulled to the local machine at all. Only the new layer is pushed together with the new image manifest.


[[Image rebase and improved remote cache support in new BuildKit.png|Open: Pasted image 20231102183033.png]]
![[Image rebase and improved remote cache support in new BuildKit.png]]
### Differences from `--link=false`

You might wonder why was the new flag added at all instead of changing all the `COPY` commands to use new semantics automatically. The reason is that it is not completely backward-compatible in some rare cases. For example, let’s say your copy command is `COPY myapp /path/to/myapp`. If the destination directory you specified in `/path/to/myapp` contained a symlink in one of the components, it would have been followed and files copied to the symlink target instead. With `--link`, all the copies are independent, and they are never allowed to see what files the destination path contained. So instead of following a symlink, `COPY --link myapp /path/to/myapp` would first always create a new directory `/path/to` and copy the file inside it.

Another case you might see is with a command like `COPY myapp /usr/bin`. Notice that the destination path does not end with a slash. Without `--link` the previous semantics would have checked if `/usr/bin` is a directory. If it was, then the file would be copied as `/usr/bin/myapp`. If it was not then the new file would have been copied to `/usr` as regular file `bin`. These kinds of checks require extracting files on disk so that their types can be verified and are not allowed with `--link`. Therefore when using `--link`, you need to make sure that the destination path does not contain a symlink and not use ambiguous destination directory detection.

The cases listed above should be quite rare and easy to fix by simple Dockerfile modifications. If you don’t rely on symlinks in your `COPY` commands, the recommendation is to always start using `--link`. The performance of linked copies should always be either better or equivalent to regular copies, and you get much better cache reuse and optimizations for your builds.

If you are interested more about the internals on `COPY --link`, it is powered by the new MergeOp feature in BuildKit’s LLB definition. You can read more about MergeOp, as well as the companion DiffOp feature that is conceptually a reverse of MergeOp from [BuildKit documentation](https://github.com/moby/buildkit/blob/v0.10.0/docs/merge%2Bdiff.md)..