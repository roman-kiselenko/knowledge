---
title: Building Docker Images The Proper Way
source: https://martinheinz.dev/blog/42
clipped: 2023-09-04
published: 2021-02-01
category: development
tags:
  - containers
read: true
---

At this point probably everybody has heard about Docker and most developers are familiar with it, use it, and therefore know the basics such as how to build a Docker image. It is as easy as running `docker built -t name:tag .`, yet there is much more to it, especially when it comes to optimizing both the build process and the final image that is created. So, in this article we will go beyond the basics and we will look at how we can influence the build process of Docker images to make it faster and to produce much slimmer and more secure images for our applications.

## Caching For Speedy Builds

Bulk of the image build time often comes from download and installation of system libraries and application dependencies. These are however not changed very often and therefore are good candidates for caching.

Starting with system libraries and tools - it's oftentimes enough to move their installation right after initial `FROM` to make sure they're cached. Regardless of which Linux distro you're using as base image, you should end up with something like this:

```
FROM ... # any viable base image like centos:8, ubuntu:21.04 or alpine:3.12.3


RUN yum install ...

RUN apt-get install ...

RUN apk add ...

```

Alternatively you could even extract all these out into separate `Dockerfile` to build your own base image. This image then can be pushed to registry, so that you and others can reuse it for multiple application. This way you don't worry about any system libraries/dependencies unless you need to upgrade them or add/remove something.

After system libraries we usually want to install application dependencies. Those could be Java libraries from Maven repositories stored in `.m2` directory, JavaScript modules in `node_modules` or Python libraries in `venv`. These will change more often then the system dependencies but not often enough to warrant a complete re-download and re-installation with every build. With badly written Dockerfile though, you will notice that cache is not going to be used even when dependencies are not modified:

```
FROM ...  # any viable base image like python:3.8, node:15 or openjdk:15.0.1


COPY . .


RUN mvn clean package

RUN pip install -r requirements.txt

RUN npm install

CMD [ "..." ]
```

Why is that? Well, the problem lies with the `COPY . .`, Docker uses cache in each step of the build until it runs into command/layer that is new/modified. In this case when we copy everything into the image - including *unchanged* list of dependencies as well as *modified* source code - Docker just goes ahead and re-downloads and re-installs all dependencies because it can't use the cache for this layer anymore because of modified source code. To avoid this, we have to copy files in 2 steps:

```
FROM ...  # any viable base image like python:3.8, node:15 or openjdk:15.0.1

COPY pom.xml ./pom.xml                    # Java
COPY requirements.txt ./requirements.txt  # Python
COPY package.json ./package.json          # JavaScript

RUN mvn dependency:go-offline -B          # Java
RUN pip install -r requirements.txt       # Python
RUN npm install                           # JavaScript

COPY ./src ./src/
```

First we add the file that lists all application dependencies and install them. If there were no changes to this file, then this will all be cached. Only then we copy rest of the (modified) sources into the image and run tests and build of the application code.

And for a little more *"advanced"* approach we do the same using Docker's *BuildKit* and it's experimental features:

```


FROM ...  # any viable base image like python:3.8, openjdk:15.0.1
COPY pom.xml ./pom.xml                    # Java
COPY requirements.txt ./requirements.txt  # Python

RUN --mount=type=cache,target=/root/.m2 mvn dependency:go-offline -B             # Java
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt   # Python
```

The above code shows how we can use `--mount` option of `RUN` command to select cache directory. This can be helpful if you want to explicitly use non-default cache location. If you want to use this feature though, you will have to include the header line specifying the syntax version (as above) and also run the build with `DOCKER_BUILDKIT=1 docker build name:tag .`. More info on the experimental features can be found in these [docs](https://github.com/moby/buildkit/blob/master/frontend/dockerfile/docs/syntax.md#run---mounttypecache).

All that was said so far applies only to local builds - for CI it's a different story and in general it's going to be different for every tool/provider, but for any of them you will need some persistent volume which will store the cache/dependencies. For example for [Jenkins](https://www.jenkins.io/doc/book/pipeline/docker/#caching-data-for-containers), you can use storage in the agent. For Docker builds running on Kubernetes - whether its using JenkinsX, Tekton or whatever else - you will need Docker daemon, that can be deployed using *Docker in Docker (DinD)*, which is Docker daemon running in Docker container. As for the build itself, you will need one pod (container) that connects to the *DinD* socket and runs `docker build`.

For demonstration purposes and to make things simple we can do that using following pod:

```
apiVersion: v1
kind: Pod
metadata:
  name: docker-build
spec:
  containers:
  - name: dind  
    image: docker:19.03.3-dind
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ''
    volumeMounts:
      - name: dind-storage
        mountPath: /var/lib/docker
  - name: docker  
    image: docker:19.03.3-git
    securityContext:
      privileged: true
    command: ['cat']
    tty: true
    env:
    - name: DOCKER_BUILDKIT
      value: '1'
    - name: DOCKER_HOST
      value: tcp://localhost:2375
  volumes:
  - name: dind-storage
    emptyDir: {}
  - name: docker-socket-volume
    hostPath:
      path: /var/run/docker.sock
      type: File
```

The above pod consists of 2 containers - one for *DinD* and one for the *image builder*. To run builds using the builder container, you can access its shell, clone some repository and run build:

```
~ $ kubectl exec --stdin --tty docker-build -- /bin/sh  
~ 
~ 
~ 
...
 => importing cache manifest from martinheinz/python-project-blueprint:flask
...
 => => writing image sha256:...
 => => naming to docker.io/library/name:tag
 => exporting cache
 => => preparing build cache for export
```

The final `docker build` uses some new options - one being `--cache-from image:tag` which tells Docker that it should use specified image from (remote) registry as a cache source. This way we can take advantage of caching even if the cached layers are not stored on local filesystem. The other option - `--build-arg BUILDKIT_INLINE_CACHE=1` - is used to write cache metadata into image when it's created. This has to be used for `--cache-from` to work, more info on this in [docs](https://docs.docker.com/engine/reference/commandline/build/#specifying-external-cache-sources).

## Slimming Them Down

It's nice to have fast builds, but if you have real *"thick"* images, it will still take a long time to pull and push them, and fat images will most likely also contain lots of libraries, tools and whatnot that make the image more vulnerable as it creates bigger attack surface.

The simplest way to make slimmer images is to use something like Alpine Linux instead of Ubuntu or RHEL based images. Another good approach is to use multi-step Docker builds, where you use one image for building (1st `FROM` command) and different, slimmer image for running the application (2nd/last `FROM`), for example:

```

FROM python:3.8.7 AS builder

COPY requirements.txt /requirements.txt
RUN /venv/bin/pip install --disable-pip-version-check -r /requirements.txt


FROM python:3.8.7-alpine3.12 as runner


COPY --from=builder /venv /venv
COPY --from=builder ./src /app

CMD ["..."]
```

The above shows that we first prepare the application and its dependencies in basic Python 3.8.7 image which is quite big at 332.88 MB. In this image we install virtual environment and libraries needed by the application. We then switch to much smaller Alpine-based image which is only 16.98 MB. To this image we copy whole virtual environment which we previously created as well as our source code. This way we end up with much smaller image with fewer layers as well as fewer unnecessary tools and binaries lying around.

One additional thing to bear in mind is number of layers we produce during every build. `FROM`, `COPY`, `RUN` and `CMD` are the four commands that create layers and at least in case of `RUN` we can easily reduce number of layers it creates by `&&`\-ing all the `RUN` commands into single one like this:

```

RUN yum --disablerepo=* --enablerepo="epel"
RUN yum update
RUN yum install -y httpd
RUN yum clean all -y


RUN yum --disablerepo=* --enablerepo="epel" && \
    yum update && \
    yum install -y httpd && \
    yum clean all -y
```

We can push it even farther and get rid of the possibly quite thick base image completely. To do that, we would use special `FROM scratch` which signals to Docker that it should use minimal base image and that next command will be first layer of final image. This is especially useful for application that run as binaries and don't require a bunch of tools to run, such as Go, C++ or Rust applications. This approach however requires that the binary is statically compiled and therefore it won't work for languages like Java or Python. An example of `FROM scratch` Dockerfiles might like so:

```
FROM golang as builder

WORKDIR /go/src/app
COPY . .

RUN CGO_ENABLED=0 go install -ldflags '-extldflags "-static"'

FROM scratch
COPY --from=builder /go/bin/app /app
ENTRYPOINT ["/app"]
```

Pretty simple, right? And with this kind of Dockerfile we can produce images that are only around 3MB!

## Locking Things Down

Speed and size efficiency are the 2 things most people will focus on while security of images becomes an afterthought. There are simple ways to lock things down in your images and to limit the attack surface available to adversaries.

The most basic recommendation that's important not just for security, but also for stability of images is to lock versions of all libraries, packages, tools and base images. If you use `latest` tag for images or you for example do not specify versions in Python's `requirements.txt` or JavaScript `package.json` you're risking that the image/library that will downloaded during build might not be compatible with the application code or it might expose your container to vulnerabilities. While you want to lock everything to specific version you should also periodically update all these dependencies to make sure you have all the latest security patches and fixes available.

Even if you try really hard to avoid any vulnerabilities in all the dependencies, there will still be some that you either miss or are yet to be fixed/discovered. So, to mitigate impact of any possible attack, it's best to avoid running containers as `root`. You should therefore include `USER 1001` in your Dockerfiles to signify that containers created from your Dockerfiles should and can run as non-root (ideally arbitrary) user. Of course this might require you to modify your application as well to choose right base image as some common base images like `nginx` require root privileges (e.g. because of privileged ports).

It's generally hard to find/avoid vulnerabilities in Docker images but it can made a little easier if the image includes only the bare minimum needed to run the application. One such image - or rather set of images - is [Distroless](https://github.com/GoogleContainerTools/distroless) made by Google. Distroless images are trimmed down to the point that they don't even have shells or package managers, which makes them much better security-wise than Debian or Alpine-based images. If you're using multi-step Docker build, then most of the time, switching to Distroless *runner* image is as simple as this:

```
FROM ... AS builder




FROM gcr.io/distroless/python3 AS runner

FROM gcr.io/distroless/base AS runner

FROM gcr.io/distroless/nodejs:10 AS runner

FROM gcr.io/distroless/cc AS runner

FROM gcr.io/distroless/java:11 AS runner



```

Apart from possible vulnerabilities in final image and its containers, we have to also thing about the Docker daemon and container runtime that we're using to build the images. So, same as with all our images, we should not allow Docker to run with `root` user, but rather use so-called *rootless* mode. There's whole guide on how one can set that up in [Docker docs](https://docs.docker.com/engine/security/rootless/), if you however don't want to mess with this kind of configuration, then you might want consider switching to `podman` which runs by default in *rootless* and *daemonless* mode - more on that in my other article [here](https://martinheinz.dev/blog/35)

## Conclusion

Containers and Docker has been around for long enough for everybody learn and use little more than just the bare minimum of the available features. The tips and examples in this article should (in my opinion) be a good stepping stone to up your Docker knowledge and improve all aspects of Docker images you use. There are however many more things outside of building Docker images one can do to improve the way we work with images and containers. This includes for example applying `seccomp` policies (see my previous [article](https://martinheinz.dev/blog/40)), limiting resources consumption using `cgroups` or possibly using completely different container runtime/engine.