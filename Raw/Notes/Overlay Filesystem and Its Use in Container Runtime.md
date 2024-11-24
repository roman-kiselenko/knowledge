---
title: Overlay Filesystem and Its Use in Container Runtime
source: https://medium.com/@jinha4ever/overlay-filesystem-and-its-use-in-container-runtime-693eba88a2a7
clipped: 2024-07-28
published: 
category: k8s
tags:
  - containers
  - docker
  - resources
read: false
---

Have you ever wondered how Container Runtime manages container images to make the best use of disk storage and offers individual file system view for each container? Today, I would like to discuss the overlay filesystem and how container runtimes leverage this type of filesystem. After reading this article, you will get a better understanding of the overlay filesystem’s role in container technology, its benefits for efficient storage utilization, and how it enables runtime isolation and independence among containers.

I created a Kubernetes cluster using GKE, and it automatically assigns an external IP to each node, allowing you to log in to the node via this IP. On your local terminal, you can generate a public and private SSH key pair for gaining access to the node by using the command `ssh-keygen`. Then, enter the following command:

gcloud compute ssh {YOUR\_NODE\_NAME} --zone {YOUR\_ZONE}

This command will automatically register your public SSH key with Google Cloud Platform (GCP) if it hasn’t been manually registered already. Therefore, when you visit GCP and navigate to the SSH Keys menu under Compute Engine, you’ll find the newly registered SSH public key listed there.

![[Raw/Media/Resources/6ac33093f10369a7acbf5a469a9b9b56_MD5.png]]

![[Raw/Media/Resources/f02ae05cda516e1c1148a249555d7b66_MD5.png]]

Now that you are logged onto the node, execute `sudo -i` to switch to the root user, then run `df -h` to display the disk space usage in a human-readable format.

![[Raw/Media/Resources/3db1a2ac19f65392dc704daa8825cc0b_MD5.png]]

You can see many mount points associated with the overlay filesystem. To take a closer look at these types of filesystems and their mount points, run `df -h | grep overlay`.

![[Raw/Media/Resources/051653be673e8c8823603abb4d10f4d1_MD5.png]]

As you can see, these mount points have much in common. They start with `/run/containerd/io.containerd.runtime.v2.task/k8s.io`, and there's some hash value going on after that, before finishing with `/rootfs`. What’s the purpose of these directories and why do they utilize an overlay filesystem? And further more, what exactly is an overlay filesystem?

When a container is running on a Kubernetes cluster, it leverages an overlay filesystem to have its own filesystem. The overlay filesystem is a technique that merges various directories into a single view with a designated mount point. An overlay filesystem requires three directories to be set up for creation: ‘lowerdir’, ‘upperdir’, and ‘workdir’. By examining the command below, you can gain a better understanding of what an overlay filesystem is:

sudo mount -t overlay overlay -o lowerdir=/path/to/lower,upperdir=/path/to/upper,workdir=/tmp/overlay /mnt/merged

The `lowerdir` is designated for read-only access and serves as the base layer. The `upperdir` is the directory where all modifications are stored, acting as the writable layer. The `workdir` is used for processing changes before they are finalized in the `upperdir` thus it is not included in the unified view. By accessing the filesystem through the `/mnt/merged` mount point, you obtain a unified view that seamlessly integrates the `lowerdir` and `upperdir`. In a containerization environment, containers should run in separate environments; they have their own file systems. They utilize overlay filesystems to manage changes made while the container is running and can also share the same base image among other containers, thereby saving a significant amount of disk space.

Let’s take one example from the captured picture from above

 /run/containerd/io.containerd.runtime.v2.task/k8s.io/501b7d056e57935a17e664733c527d6db1b20c6fda186dc1d6bfa4bcac0ee08a/rootfs

To get straight to the conclusion, the mount point specified is the root filesystem of a container, identified by the container ID ‘501b7d056e57935a17e664733c527d6db1b20c6fda186dc1d6bfa4bcac0ee08a’, and it is managed by containerd within a Kubernetes (k8s.io) environment. If you navigate to the directory `/run/containerd/io.containerd.runtime.v2.task/k8s.io/501b7d056e57935a17e664733c527d6db1b20c6fda186dc1d6bfa4bcac0ee08a`, you will be able to see essential files and subdirectories for managing a container within a Kubernetes environment, managed by containerd. Crucially, the `rootfs` directory exists as well, serving as the container's root filesystem.

![[Raw/Media/Resources/ab3f0e4788bd9cf5a6794110d57736f9_MD5.png]]

If you want to understand why `rootfs` is the root directory of the container, you can run the following command:

cat config.json | grep root

![[Raw/Media/Resources/81140953fef745542df6741e1138d34a_MD5.png]]

This will show you that the root path is configured as `rootfs`. Therefore, `/run/containerd/io.containerd.runtime.v2.task/k8s.io/501b7d056e57935a17e664733c527d6db1b20c6fda186dc1d6bfa4bcac0ee08a/rootfs` serves as the mount point, offering a unified view of the readable and writable directories of the root file system for this specific container. Listing this directory will reveal the files and directories contained within.

![[Raw/Media/Resources/f17c876924a68113735ae7c324f590d8_MD5.png]]

Now, I would like to show you the composition of this container’s root filesystem. You can see that by running the following command:

cat /proc/mounts | grep 501b7d056e57935a17e664733c527d6db1b20c6fda186dc1d6bfa4bcac0ee08a

  
overlay   
/run/containerd/io.containerd.runtime.v2.task/k8s.io/501b7d056e57935a17e664733c527d6db1b20c6fda186dc1d6bfa4bcac0ee08a/rootfs   
overlay   
rw,relatime,  
lowerdir=  
 /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/5/fs  
:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/4/fs  
:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/3/fs  
:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/2/fs  
:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/1/fs,  
upperdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/77/fs,  
workdir=/var/lib/containerd/io.containerd.snapshotter.

By definition, the mount point `/run/containerd/io.containerd.runtime.v2.task/k8s.io/501b7d056e57935a17e664733c527d6db1b20c6fda186dc1d6bfa4bcac0ee08a/rootfs` offers a unified view of the following directories:

-   `/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/5/fs`
-   `/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/4/fs`
-   `/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/3/fs`
-   `/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/2/fs`
-   `/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/1/fs`
-   `/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/77/fs`

This list illustrates the layered structure that composes the root filesystem of this specific container. The image below shows the result of listing all the directories mentioned above.

![[Raw/Media/Resources/d4489489bced54e0fd542e2b43633da6_MD5.png]]

Below is the unified view once again.

![[Raw/Media/Resources/f17c876924a68113735ae7c324f590d8_MD5.png]]

So, by separating directories into writable and readable categories, each container can have its own unique filesystem. While they are running, containers can write whatever they want in their configured upper directory, and it won’t affect the lower directories, which are read-only. Thus, containers with the same image would share the same lower directories but will have their own upper directory so that they can be independent of one another.

Below are the configurations of two containers employing an overlay filesystem, both using the same image.

  
overlay   
/run/containerd/io.containerd.runtime.v2.task/k8s.io/993d6f7f729c0087eb74aca5c78fa5a95507abf151171ed0c06e0e3430a2060c/rootfs   
overlay rw,relatime,  
lowerdir=  
 /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/316/fs  
:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/315/fs  
:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/314/fs  
:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/313/fs  
:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/312/fs  
:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/311/fs  
:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/310/fs,  
upperdir=  
/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/318/fs,  
workdir=  
/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/318/work 0 0

  
overlay /run/containerd/io.containerd.runtime.v2.task/k8s.io/c78356253c004b7e23b205145930e23e0d9b6c32c1d587fb4d60b60343dafc64/rootfs   
overlay rw,relatime,  
lowerdir=  
 /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/316/fs  
:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/315/fs  
:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/314/fs  
:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/313/fs  
:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/312/fs  
:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/311/fs  
:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/310/fs,  
upperdir=  
/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/317/fs,  
workdir=  
/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/317/work 0 0

As I explained earlier, containers created from the same image indeed share the same read-only layers (lower directories, from snapshot 310 to 316) in an overlay filesystem. This allows for efficient use of disk space and resources, while also ensuring runtime isolation and independence among containers.

In conclusion, I hope this discussion has provided you with a better understanding of the importance and functionality of the overlay filesystem within container runtime environments. Since it’s been a bit lengthy, I’ll say goodbye for now!