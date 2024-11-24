---
title: "Kubernetes Services : Achieving optimal performance is elusive"
source: https://cloudybytes.medium.com/kubernetes-services-achieving-optimal-performance-is-elusive-5def5183c281
clipped: 2024-05-12
published: 
category: network
tags:
  - network
  - networking
read: false
---

This blog is to share with readers the experiences and experimentation results while deploying an on-premise Kubernetes service project.

Due to certain management policies at work, it was decided to deploy this particular app as a cloud-native on-prem service. The DevOps guys jumped into this fantastic opportunity. All good — app was containerized , deployed using a [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/). We chose a modest flannel-based networking model which did a great job. Kill one pod and another spawned automatically and networking worked flawlessly. At the same time, Dev team finished building the front-end client. Access to this service seemed easy : four nodes deployed, hence use four node-IP/node-ports combinations pairs to get access to this service. Nobody really bothered about introducing a real LB at this stage, since the consensus was that it could be easily handled at the client-side app.

![[Raw/Media/Resources/fa934471ea5718651f1c0e15a38affd8_MD5.jpg]]

It was soon realized what a bad design call it was. At times, the node IP addresses changed for some reason, while at other times, some nodes were heavily loaded and yielded poor performance. The front-end client was soon transforming into an advanced load-balancing programming challenge. Luckily, better sense prevailed and someone suggested using service-type [load-balancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) like hyper-scalers do. [MetalLB](https://metallb.universe.tf/) was a natural choice at the time. We already used kube-proxy in IPVS mode needed by MetalLB as part of our flannel setup. (For starters, [kube-proxy](https://kubernetes.io/docs/concepts/overview/components/#kube-proxy) is the duct-tape of K8s world. By default, it uses iptables which is notoriously difficult to scale but we used ipvs mode which supposedly has better numbers.)

![[Raw/Media/Resources/ffca6a6bd6eb362b22f0605fd56f80ec_MD5.jpg]]

The Dev team were quite satisfied not having to deal with load-balancing nightmare themselves. The so-called VIP of this service was fed to a local DNS server and they could then use named service to access the cluster. All good until some users kept complaining about sluggish performance at random intervals. If the same service was run as a monolithic app in a bare-metal server, the users got satisfactory performance each time. So, definitely something was wrong and amiss.

It was really high time to look into the details and figure out what we might have overlooked. We setup a bare-bones K8s setup (for debugging) which had a single master and a single worker with only one instance (pod) of our application. The root-cause was soon discovered — Performance was always bad when VIP and selected LB end-point ended up in different nodes. VIP is usually assigned to a node based on Load-Balancer’s internal logic and this node serves all the incoming requests. The end-point Pods are simply selected on a round-robin basis or other policies as per IPVS rules. VIPs are [floating](https://www.digitalocean.com/blog/floating-ips-start-architecting-your-applications-for-high-availability) in nature depending on node/pod fail-overs.

![[Raw/Media/Resources/bc5c45fafa57c9150b7ed36a081790cc_MD5.png]]

MetalLB — Traffic via intermediate master-node

![[Raw/Media/Resources/4852f1fba3dde76eb86e6dcf9f324ea4_MD5.png]]

MetalLB — Traffic via intermediate master-node (Perf)

![[Raw/Media/Resources/0b30d04d8efd5396952d2f8a2d24ac27_MD5.png]]

MetalLB — Traffic directly to node where Pod is

![[Raw/Media/Resources/c96e17ff343c658ed27806ddaafbaf59_MD5.png]]

MetalLB — Traffic directly to node where Pod is (Perf)

Simple [iperf](https://iperf.fr/) test showed around ~80% (4Gbps -> 600 Mbps) drop in performance based on how Pods are placed and selected by LB.

Back to drawing board !! How about replacing flannel with some other CNI which has in-built service-type LB support ? Flannel served us well and its simplicity was perfect for our org. Why change something for no fault of its own. Frankly, at this time we needed something to compare against and draw some conclusions. Long story short, we decided to give [LoxiLB](https://github.com/loxilb-io/loxilb) a try. I experimented with it before for some blogs and had a relatively positive experience. Nonetheless no other option seemed viable enough. Overall, we chose LoxiLB’s [in-cluster](https://www.loxilb.io/post/k8s-nuances-of-in-cluster-external-service-lb-with-loxilb) mode, which closely resembled our earlier setup. Frankly, we were pleasantly surprised at the results. LoxiLB is based on [eBPF](http://ebpf.io/) and it seemingly is able to bypass some bottlenecks encountered by MetalLB (IPVS).

![[Raw/Media/Resources/a732e1510d71208e5403ea1ac4d2d9d3_MD5.png]]

LoxiLB — Traffic via intermediate master-node

![[Raw/Media/Resources/073846087be861c8bb9079980533dfb7_MD5.png]]

LoxiLB — Traffic via intermediate master-node (Perf)

![[Raw/Media/Resources/f10d371de035ac742bc8bf55a30d40d0_MD5.png]]

LoxiLB — Traffic directly to node where Pod is

![[Raw/Media/Resources/e4d38404b0305f346b6080055a743e2e_MD5.png]]

LoxiLB — Traffic directly to node where Pod is (Perf)

There were some surprising finds here. Firstly, the performance jumped from ***~600Mbps to 3Gbps*** in the worst case compared with MetalLB. Secondly, even in the optimal scenario, there was a gain of ***~1Gbps*** as compared to MetalLB. Lastly, performance drop was still seen depending on how Pods were placed and selected but overall improvement of ***~70%*** in system performance was achieved.

MetalLB is an awesome project but under-the-hood it uses Linux kernel’s [ipvs](https://en.wikipedia.org/wiki/IP_Virtual_Server) as the actual load-balancer. ipvs was developed as an alternative to using [iptables](https://en.wikipedia.org/wiki/Iptables) for load-balancing use-cases. But, eBPF is a game-changer for sure.

This post is not about declaring a winner in LoxiLB but rather how to trouble-shoot and use the right tools for one’s particular use-case(s). Transition to a cloud-native architecture does not automatically guarantee amazing performance and availability. Sound system-design principles are a must right from the beginning. Please visit this [repo](https://github.com/codeSnip12/cnlblab) and follow the instructions to recreate this experimental test-setup.