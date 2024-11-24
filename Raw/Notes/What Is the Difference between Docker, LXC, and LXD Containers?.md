---
title: What Is the Difference between Docker, LXC, and LXD Containers?
source: https://kodekloud.com/blog/what-is-the-difference-between-docker-lxc-and-lxd-containers/#
clipped: 2023-09-04
published: 2022-08-22
category: development
tags:
  - containers
  - docker
read: false
---

[Docker](https://kodekloud.com/blog/tag/docker/)

![Difference between Docker, LXC, and LXD Containers](https://kodekloud.com/blog/content/images/size/w2000/2022/11/What-Is-the-Difference-between-Docker--LXC--and-LXD-Containers.png)

Imagine you're a developer writing a cool app for Linux servers. But some people run Ubuntu 20.04 on their servers. Others run 22.04. Others run an entirely different operating system, such as Red Hat Enterprise Linux. So how do you make sure your app runs well on each operating system?

Normally, developers make multiple versions of their apps. One version that runs on older Ubuntu, one that runs on the newer Ubuntu, one that works with Red Hat Enterprise Linux, and so on. There need to be small adaptations to ensure the app is compatible with each operating system. This can be tedious.

This problem is exactly what containerization came to solve. Containers provide an efficient way to package and deploy an application or OS, allowing them to run consistently and reliably across different environments.

This article will discuss Docker, LXC, and LXD container technologies and their use cases.

## What is Docker?

Docker containers are **a type of containerization technology that allows developers to package an application and all of its dependencies into a single, lightweight, and portable container**. Each container runs as an isolated process on the host system, with its own file system and network interface, allowing for consistent and reliable application deployment across different environments.


An application can depend on other programs, other files, system libraries, and so on. These are called **dependencies**. Without a container solution like Docker, developers need to look at what programs and system libraries they have available on each operating system. And then, they need to make sure that their app works with those programs and libraries.

Here's an example. Imagine the app needs "*some\_library, version 1.3*". Ubuntu 22.04 has that version. But Ubuntu 20.04 has "*some\_library 0.9*". Oops, that version is too old. Now, the developer has to adapt his app for Ubuntu 20.04 so that it can work with this old library. But with Docker, they can go a different route.

They take their app and drop it inside a container. Then, they take all **dependencies**, every extra program, library, or file that their app requires. They package these nicely and also drop them inside that container. In our imaginary scenario, they can simply pick up "*some\_library 1.3*" and drop it in there. Now, this **container includes everything that the application needs** to be able to run. So it doesn't need to rely on what the operating system provides anymore.

With the perfect container created, with all the ingredients inside, the developer's job is done! Now, they can ship that app to the whole world. It will work on Ubuntu, openSUSE, Red Hat Enterprise Linux, and almost any Linux operating system without requiring any changes.

So, we can conclude that **Docker is basically a very small box where we can drop an application and all of its dependencies**. Otherwise, it contains the *environment* that it needs to be able to run.

Also, a Docker container isolates this application from the rest of the system. It's very important to note that **Docker is designed to isolate a single application inside a container**.

Lastly, Docker containers are designed to be highly scalable, allowing for easy deployment and management of applications in cloud-based environments. They are also highly configurable, with a wide range of options for customizing their behavior and resource usage.

Okay, so we have a pretty cool solution to develop applications that can be shipped to almost any Linux server configuration. Let us now look at LXC.

## What is LXC?

Let's think about an operating system. What are its components? First of all, it has a bootloader. This helps the computer find and load the operating system into memory. It also has a kernel that can initialize and control the computer's hardware. And it has many applications that let you do useful work on that computer. These components, and others, make up the operating system's *environment*. In simple terms, it has stuff that helps three different entities:

1.  It helps the computer boot up and use its hardware properly.  
    
2.  It helps programs do their work by providing the required libraries or whatever is needed.  
    
3.  It helps human beings use that computer and do their work.

Some operating systems skip the third part if human interaction is not needed. For example, this can be the case for automated electronic systems.

In our LXC discussion, the second part is the most important. The **operating system provides the environment that programs need to do their work**. For example, programs need the kernel to be able to use computer memory. They can't reserve system memory if the kernel doesn't help them do that. So, once again, we can say that the programs can have some dependencies. Notice how we mentioned two keywords: *environment* and *dependencies*. We used the same words when we discussed Docker.

Now, back to Docker. Its containers do not encapsulate an entire operating system. It doesn't need a kernel, a bootloader, or any other complex stuff. It just needs a very tiny collection of files. Maybe a few programs and libraries so that the app inside can do its work. **Docker includes a very minimal environment**. This makes Docker containers *small, easy to move around, and fast to start up*.

But what if we need an entire operating system in a container? The first thing that may come to mind is a virtual machine. But virtual machines are rather hard to create. First, we need to assign virtual resources to that virtual machine. We need to reserve a little bit of memory, a little bit of disk space, and some CPU cores. Then, we need to download an operating system. Then, we need to install it in that virtual machine.

It may be easy to create a virtual machine on the personal computer right in front of you. But it's much harder to do that on a remote server, somewhere on the Internet. And it becomes even harder, considering you have to do this through commands.

Furthermore, most servers we can rent online are actually virtual these days. The compute instances on the cloud are virtual machines. A VPS (virtual private server) is also, obviously, virtual. So, creating a virtual machine inside a virtual machine is tricky. Most cloud service providers don't even allow you to do that. Then what can we do?

Well, LXC, short for Linux Container, saves the day. It makes it very easy to run containers with an entire operating system inside. Now, if we recap, we can see the **differences between Docker and LXC**:

-   **Docker is designed to isolate ONE application in ONE container.** So, it allows you to **run multiple isolated applications** on a server.  
    
-   **LXC is designed to isolate ONE operating system in ONE container.** So, it allows you to **run multiple isolated operating systems** on a server. It's almost like LXC lets you run many small, imaginary computers on that server. And each imaginary small computer has its own separate operating system.

This sounds like a pretty cool idea. But you may ask yourself: "Why would I want other operating systems running on top of my main operating system?" Well, here's an example.

Imagine how easy it would be to do this with LXC. On a single server, we could launch a bunch of Linux Containers. You click that you want to try out commands on Ubuntu 22.04, and boom! We quickly launch a container with Ubuntu 22.04 inside. Another user comes along. We launch another separate Ubuntu 22.04 container. And now, both users can do their work in their own isolated boxes without affecting each other. When you click that you've finished testing commands, we can delete your LXC container.

So, **with Docker, you can run services that rely on one application**. And **with LXC, you can run services that rely on one operating system**.

Looking to gain or polish your Linux skills? Check out our [Linux Basics Course & Labs](https://kodekloud.com/courses/the-linux-basics-course/?ref=kodekloud.com).

## What is LXD?

LXD still uses LXC behind the scenes. But it greatly **extends LXC's functionality**. It both improves LXC's existing functions and adds new capabilities on top of it. Here are some examples of what it brings to the table:

-   LXD improves isolation between LXC containers and the rest of the system. It makes LXC containers a bit more secure, trying to ensure a rogue container doesn't affect the rest of the system.  
    
-   LXD allows you to migrate running containers from one server to another server. That is, you can transfer the entire container from one place to another place while it's still running. Once the transfer is complete, it should continue running exactly from where it left off. For example, imagine a video is rendering slowly inside that container. It got all the way to 30%. But it takes too long. So, you decide to migrate the running container to a much more powerful server. Once the migration is complete, the container should continue rendering from 30%.  
    
-   LXD allows you to limit the resources used by each container. Remember the example with one LXC container for each user that needs to test commands in a Linux environment? Well, LXC containers are not virtual machines. So, they don't have fixed reserved memory like virtual machines do. Each container can access all the memory on a server. If one container uses up all the memory, nothing is left for the other users. So, it can ruin everyone's experience. LXD allows you to establish that each container can only use 512MB of memory or something like that. Or you can limit how much CPU time each container can use.

## Conclusion

There are many more features like this. The thing is, there is rarely a reason to not use LXD. So if you have that option, just install LXD directly. The LXC dependencies will be pulled in automatically.

Hopefully, this clears up when you should use Docker containers and when you should use LXC containers. Maybe you even get some cool ideas about how to creatively use LXC to build an interesting online service.

More on containerization:

-   [Virtualization Vs Containerization: 6 Key Differences](https://kodekloud.com/blog/virtualization-vs-containerization/)
-   [Docker vs. Containerd: A Quick Comparison (2023)](https://kodekloud.com/blog/docker-vs-containerd/)
-   [Docker Containerization: Key Benefits and Use Cases](https://kodekloud.com/blog/docker-containerization/)