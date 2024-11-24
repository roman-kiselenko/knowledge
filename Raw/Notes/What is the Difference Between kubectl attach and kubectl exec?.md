---
title: What is the Difference Between kubectl attach and kubectl exec?
source: https://adil.medium.com/what-is-the-difference-between-kubectl-attach-and-kubectl-exec-148dab1e2aa3
clipped: 2023-10-06
published: 
category: k8s
tags:
  - kubectl
  - development
read: false
---

![[Raw/Media/Resources/91cf4d1f4161addd47bc5319c5d883e8_MD5.jpg]]

Photo by [Anne Nygård](https://unsplash.com/@polarmermaid?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral)

Most likely, you are already familiar with `kubectl exec` . You may log in to a container running inside a pod:

*hello-pod.yaml*

\---  
apiVersion: v1  
kind: Pod  
metadata:  
  name: hello-pod  
spec:  
  containers:  
  \- image: ailhan/control:hostname  
    name: hello-container

**Apply:**

![[Raw/Media/Resources/916c4a758adae0b3e8194931441deb39_MD5.png]]

There is another command to interact with a Pod: `kubectl attach`

You can access the container’s stdin and stdout via kubectl attach.

An example pod:

*00-pod-datetime.yaml*

\---  
apiVersion: v1  
kind: Pod  
metadata:  
  name: print-datetime  
spec:  
  containers:  
  \- image: ailhan/control:datetime  
    name: print-datetime-container

**Apply:**

![[Raw/Media/Resources/c09382bf3961f89da180a3e8b5993143_MD5.png]]

We can see the logs. Each second, the date and time are written to the standard output. But as of yet, we are unable to engage with the main process of the container directly.

You can **attach** your terminal to the main process of the container inside the pod

**kubectl attach:**

![[Raw/Media/Resources/3f5947b836b69cd22d7d1c8c806c75df_MD5.png]]

I’ve attached my terminal to the first container of the `print-datetime pod.`

I had to send CTRL+C. Otherwise, I would see the date and time every second in my terminal.

At the beginning of the article, I said the container will handle your CTRL+C and CTRL+Z interrupts. But the container doesn’t seem to catch my interrupt.

Because we have to make it clear that stdin and tty of the container must be accessible.

*01-pod-datetime.yaml*

\---  
apiVersion: v1  
kind: Pod  
metadata:  
  name: print-datetime-with-stdin  
spec:  
  containers:  
  \- image: ailhan/control:datetime  
    name: print-datetime-container  
    tty: true  
    stdin: true

**Apply:**

![[Raw/Media/Resources/d4a2ae71d11e4c3f8f6ae75a29f6e127_MD5.png]]

Oops! My terminal won’t connect to the pod I just made. Because I set `tty` and `stdin` variables in the yaml file.

I should attach my terminal to the pod with `-it` parameters:

![[Raw/Media/Resources/81c2e52f21b8411ce8c3dd6567c56d78_MD5.png]]

As I said at the beginning of the article, the script running within the container handles CTRL+C and CTRL+Z interrupts. Since I could access the container’s stdin, I could send interrupts to the container itself.

*02-pod-datetime-hostname.yaml*

\---  
apiVersion: v1  
kind: Pod  
metadata:  
  name: print-datetime-hostname-with-stdin  
spec:  
  containers:  
  \- image: ailhan/control:datetime  
    name: print-datetime-container  
    tty: true  
    stdin: true  
  \- image: ailhan/control:hostname  
    name: print-hostname-container  
    tty: true  
    stdin: true

**Apply:**

![[Raw/Media/Resources/1ef31f0c58566609ec9985d052e8d0f9_MD5.png]]

Two containers are running in the pod.

Try to attach to the pod:

![[Raw/Media/Resources/4927910c068b2b05ee00c28390c5292b_MD5.png]]

The terminal is attached to the first container in the pod. I can attach my terminal to the second container via the `-c` parameter

![[Raw/Media/Resources/10058ad1319eef5c9402c978dfaaa1ab_MD5.png]]