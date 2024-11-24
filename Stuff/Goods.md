## <mark style="background: #BBFABBA6;">Documentation</mark>

* [ko: Easy Go Containers](https://ko.build/)
* [Unix-like pipelines for Go](https://labix.org/pipe)
* [https://buildpacks.io/](https://buildpacks.io/)
* [Container Object Storage Interface (COSI) driver on Kubernetes.](https://container-object-storage-interface.github.io/docs/write-driver/introduction)
* [Concurrency patterns in Go](https://github.com/lotusirous/go-concurrency-patterns) 
* [Debug Golang Memory Leaks with Pprof](https://www.codereliant.io/memory-leaks-with-pprof/) 
* [A light library to allow changing pod log level without restarting the pod](https://github.com/gianlucam76/pod-log-level)
* [Design workflows of slog handlers](https://github.com/samber/slog-multi) 
* [Extract files from any kind of container formats](https://github.com/onekey-sec/unblob)
* [How To Secure A Linux Server](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server) 
* [System Programming](https://lowlvl.org/) 
* [Simple webservice to validate data with CUE](https://gist.github.com/owulveryck/8af03b6711c84f6672efc3e8b979a536)
* [Gateway API](https://github.com/kube-vip/kube-vip-gateway-api)
* [How to (and how not to) design REST APIs](https://github.com/stickfigure/blog/wiki/How-to-%28and-how-not-to%29-design-REST-APIs?utm_source=hackernewsletter&utm_medium=email&utm_term=code)
* [Structured logging](https://github.com/golang/go/discussions/54763)


## <mark style="background: #ADCCFFA6;">Tools</mark>

* [EIZO MONITOR TEST](https://www.eizo.be/monitor-test/)
* [Secure Password Generator](https://passwordsgenerator.net/)
* [EMBA - The firmware security analyzer](https://github.com/e-m-b-a/emba)
* [Firmware Analysis Tool](https://github.com/ReFirmLabs/binwalk)
* [Bad Duck](https://www.kitploit.com/2018/04/bad-ducky-rubber-ducky-compatible-clone.html)
* [OFRAK: unpack, modify, and repack binaries.](https://github.com/redballoonsecurity/ofrak)
* [Bootloader basics](https://notes.eatonphil.com/bootloader-basics.html)
* [Install Arch Linux](https://sidsbits.com/Arch-Install/#Motivation) 

## <mark style="background: #D2B3FFA6;">Kubernetes</mark>

* [About An example of how to build a custom kubectl sidecar container](https://github.com/thockin/kubectl-sidecar)
* [Deprecated API Migration Guide | Kubernetes](https://kubernetes.io/docs/reference/using-api/deprecation-guide/)
* [Kubetools - Curated List of Kubernetes Tools](https://github.com/collabnix/kubetools)
* [Guides and API References for Kubectl and Kustomize](https://kubectl.docs.kubernetes.io/guides/) 
* [Kubernetes Cluster API](https://cluster-api.sigs.k8s.io/) 
* [GitHub - boz/kail: kubernetes log viewer](https://github.com/boz/kail)
* [A curated list of awesome Kubernetes tools and resources](https://github.com/tomhuang12/awesome-k8s-resources)
* [GitHub - stefanprodan/podinfo: Go microservice template for Kubernetes](https://github.com/stefanprodan/podinfo) 
* [[improved-alerting-with-prometheus-and-alertmanager.pdf]]
* [Awesome Prometheus alerts | Collection of alerting rules](https://samber.github.io/awesome-prometheus-alerts/sleep-peacefully.html)
* [Bootstrap Kubernetes the hard way on Google Cloud Platform. No scripts.](https://github.com/DushanthaS/kubernetes-the-hard-way-on-proxmox?tab=readme-ov-file)
* [A guide to setting up a production-like Kubernetes cluster on a local machine](https://github.com/ghik/kubernetes-the-harder-way)
* [Kubernetes Easy Engine (k8e)](https://getk8e.com/docs/concepts/introduction/)
* [bootstrap K3s over SSH in < 60s 🚀](https://github.com/alexellis/k3sup)

## Design

* [Free Vector Illustrations For Your Projects](https://www.freeillustrations.club)
* [Free Icons](https://iconbuddy.app)
* [Excalidraw — Collaborative whiteboarding made easy](https://excalidraw.com/)
* [ambiph.one](https://ambiph.one/)

## RG35XX

* [Setup galic and sd card](https://github.com/skyzyx/rg35xx-garlicos-macos-instructions/blob/main/docs/installing-garlicos-single-card.en_us.md)
* [MinUI](https://github.com/shauninman/union-minui)
* [Build](https://www.reddit.com/r/MiyooMini/comments/10bpdcl/attempting_to_alter_miniui_for_personal_use/)
```shell
# cross linker 
$ sudo apt-get install -qq gcc-arm-linux-gnueabihf
# cross compiled `std` crate 
$ rustup target add armv7-unknown-linux-musleabihf 
$ cargo new --bin hello && cd $_ 
# point Cargo to the right linker 
$ export CARGO_TARGET_ARMV7_UNKNOWN_LINUX_MUSLEABIHF_LINKER=arm-linux-gnueabihf-gcc
# if C dependencies are involved 
$ export CC_armv7_unknown_linux_musleabihf=arm-linux-gnueabihf-gcc
$ cargo build --target armv7-unknown-linux-musleabihf
```

### Develompent

* https://github.com/psk907/music-player-framework-in-c/blob/master/main.c
* https://lazyfoo.net/SDL_tutorials/lesson06/index.php 
* https://github.com/AgelessArchangel/InputTester-for-RG35xx-Garlic-OS

## Firecracker
[Github](https://github.com/firecracker-microvm/firecracker)

https://firecracker-microvm.github.io/

Secure and fast microVMs for serverless computing.

Firecracker Manager Go example https://gist.github.com/jvns/9b274f24cfa1db7abecd0d32483666a3

Blog articles:

* [How to run end-to-end tests 10x faster with firecracker](https://webapp.io/blog/github-actions-10x-faster-with-firecracker/)
* [Firecracker: start a VM in less than a second](https://jvns.ca/blog/2021/01/23/firecracker--start-a-vm-in-less-than-a-second/)
* [How AWS Firecracker works: a deep dive](https://unixism.net/2019/10/how-aws-firecracker-works-a-deep-dive/)

Usefull links:

[The Definitive KVM (Kernel-based Virtual Machine) API Documentation](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/virt/kvm/api.txt?h=v5.3)
[Types_of_Interrupts](https://wiki.osdev.org/IRQ#Types_of_Interrupts)
[https://github.com/ChrisTitusTech/winutil](https://github.com/ChrisTitusTech/winutil)
