---
title: "Security Containers: Control Groups"
source: https://medium.com/@lessandro.ugulino/security-containers-control-groups-6a50dc5a6069
clipped: 2024-03-20
published: 
category: security
tags:
  - containers
read: false
---

![[Raw/Media/Resources/63fc7c56c73f77a5fd8c7a5cf4cb7cff_MD5.png]]

Today, I’d like to talk about one of the fundamental building blocks that are used to make containers: **control groups**, as frequently known as **cgroups**.

Basically, **cgroups** limit the resources, such as memory, CPU, and network input/output, that a group of processes can use.

> In terms of security, **cgroups** can ensure that one process can’t affect the behaviour of the other process by hogging all the resources, for example, using all the CPU or memory to starve other applications.

`cgroup` “How much you can use”  
`namespace` “How much you can see”

Let’s see how *cgroups* are organized.

*Cgroup* controller manages the hierarchy for each type of resource. Any Linux process is a member of one *cgroup* of each type.

The Linux kernel communicates information about *cgroups* through a set of pseudo-filesystems that usually is located at `/sys/fs/cgroup`.

![[Raw/Media/Resources/df1256bb6bc70e12c46e9735ffda0171_MD5.png]]

In terms of management, it’ll involve reading and writing to the files and directories within these hierarchies.

The below image describes the memory *cgroup*.

![[Raw/Media/Resources/64797dac82399217811fa18863254feb_MD5.png]]

Some of these files are written by the Kernel and others can be modified. There’s no specific way to tell which are parameters and which are informational without consulting the [documentation](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/memory.html). The purpose of some of these files’ names is intuitive, for example, **memory.limit\_in\_bytes** holds a writable value that sets the amount of memory available to processes in the group; **memory.max\_usage\_in\_bytes** reports the max memory usage within the group.

If you want to limit memory usage for a process, you’ll need to create a new *cgroup* and then assign the process to it.

When you create a subdirectory inside this memory directory, you’re creating a *cgroup*, and the kernel will automatically populate the directory with the various files that represent parameters and statistics about the *cgroup*:

![[Raw/Media/Resources/348fe84de78401439ad54dbd0b2b2ef1_MD5.png]]

As you can see, some of these files hold parameters that’ll define the limits (example: *memory.limit\_in\_bytes*) and others communicate statistics (example: *memory.usage\_in\_bytes*) about the current use of resources in the control group.

The container [runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) will create new *cgroups* when you start a container. You can use the [lscgroup](https://linux.die.net/man/1/lscgroup) command to list all cgroups.

Let’s see the difference in memory when you start a container.

Take a snapshot of the memory cgroups:

`lscgroup memory:/ > before.memory`

Start a container:

`docker run --name nginx -p 80:80 -d nginx`

Take another snapshot of the memory cgroups and compare both:

`lscgroup memory:/ > after.memory`

`diff before.memory after.memory`

![[Raw/Media/Resources/b7b5f8a50c28950b228d43e956609a3c_MD5.png]]

While the container is still running, we can inspect the *cgroup* from the host:

`ls docker/d46b4e91ea4f13aa86134306e364fb5906f184ade224911ebd52e4ff7f2fbd61`

![[Raw/Media/Resources/e5f7b7529af185990a725af58b5ab8b9_MD5.png]]

The list inside the container is available from the `/proc` directory:

![[Raw/Media/Resources/5f813a3c34ae8ee2b682df1dcaeba1e1_MD5.png]]

Once you have a *cgroup*, you can modify parameters within it by writing to the appropriate files.

The file **memory.limit\_in\_bytes** will show how much memory is available to the *cgroup*.

![[Raw/Media/Resources/4f3380e824ec89d1ad3df396999f744e_MD5.png]]

By default the memory isn’t limited, this number represents all the memory available to the virtual machine I’m using to run this container.

As there is no limit for this parameter, a process is allowed to consume unlimited memory, or it could be compromised by a [resource exhaustion attack](https://en.wikipedia.org/wiki/Resource_exhaustion_attack) that takes advantage of a memory leak to deliberately use as much memory as possible. You can reduce this kind of attack and ensure that other processes can carry on as normal by setting limits on the memory and other resources.

You can restrict the memory by running the below command or for [Kubernetes](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/).

`docker run -m 512m -it --rm -d -p 8080:80 --name web nginx`

`-m: memory`

Now you’ll find that the *memory.limit\_in\_bytes* parameter is approximately what you configured as the limit.

![[Raw/Media/Resources/d2c93d10fa3d5dccf4b78924ce3fc05b_MD5.png]]

Similar to setting resource limits, assigning a process to a *cgroup* is a simple matter of writing its process ID into the *cgroup.procs* file for the *cgroup*.

![[Raw/Media/Resources/f66b160fa509f805186fefb65f6245c1_MD5.png]]

The below command will write the process ID (1430, it’s the process ID of a shell)

![[Raw/Media/Resources/7a1bbed786bff14c2fd6d9de73bfb270_MD5.png]]

The shell is now a member of a *cgroup*, with its memory limited to a little under **100kB**. When I run `ls` command the process gets killed when it attempts to exceed the memory limit.

![[Raw/Media/Resources/d31406998325f3ea73664dd38cb627a7_MD5.png]]

Since 2016 there has been version 2 of *cgroups*. The biggest difference is that in *cgroups* v2 a process can’t join different groups for different controllers. In v1 a process could join `/sys/fs/cgroup/memory/mygroup` and `/sys/fs/cgroup/pids/yourgroup`. In v2 the process joins `/sys/fs/cgroup/ourgroup` and is subject to all the controllers for `ourgroup`.

Akihiro Sudo summarized the new version [here](https://medium.com/nttlabs/cgroup-v2-596d035be4d7).