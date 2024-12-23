---
title: Cgroups - Deep Dive into Resource Management in Kubernetes
source: https://martinheinz.dev/blog/91
clipped: 2024-12-23
published: 2023-02-20
category: development
tags:
  - containers
  - os-process
read: false
---

There's a lot of *"magic"* that happens behind the scenes to make whole Kubernetes work. One of those is resource management and resource allocation done by Linux *cgroups*.

In this article we will take a deep dive into what cgroups are, how Kubernetes uses them to manage Node resources, and how we can take advantage of them beyond setting resource requests and limits on Pods.

## What are Cgroups?

First things first - *What are cgroups anyway?* - *[Control Groups](https://docs.kernel.org/admin-guide/cgroup-v1/cgroups.html)*, or cgroups for short, are a Linux kernel feature that takes care of resource allocation (CPU time, memory, network bandwidth, I/O), prioritization and accounting (aka how much is container using?). Additionally, besides being a Linux primitive, cgroups are also a building block of containers, so without cgroups there would be no containers.

As the name implies - the cgroups are *groups*, so they group processes in parent-child hierarchy, which forms a tree. So, if - for example - parent cgroup is assigned `128Mi` of RAM, then sum of RAM usage of all of its children cannot exceed `128Mi`.

This hierarchy lives in `/sys/fs/cgroup/`, which is the cgroup filesystem (`cgroupfs`). There you will find sub-trees for all Linux processes. Here we're interested in how cgroups impact scheduling and resources assigned/allocated to our Kubernetes Pods, so the part we care about is `kubepods.slice/`:

```
/sys/fs/cgroup/
└── kubepods.slice/
    ├── kubepods-besteffort.slice/
    │   └── ...
    ├── kubepods-guaranteed.slice/
    │   └── ...
    └── kubepods-burstable.slice/
        └── kubepods-burstable-pod<SOME_ID>.slice/
            ├── crio-<CONTAINER_ONE_ID>.scope/
            │   ├── cpu.weight
            │   ├── cpu.max
            │   ├── memory.min
            │   └── memory.max
            └── crio-<CONTAINER_TWO_ID>.scope/
                ├── cpu.weight
                ├── cpu.max
                ├── memory.min
                └── memory.max
```

All the Kubernetes cgroups are located in `kubepods.slice/` subdirectory, which has further `kubepods-besteffort.slice/`, `kubepods-burstable.slice/` and `kubepods-guaranteed.slice/` subdirectories for each [QoS (*Quality-of-Service*)](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/) type. Under these directories you will find directories for each Pod and inside those, further directories for each container.

At each level there are files such as `cpu.weight` or `cpu.max` that specify how much of the particular resource - e.g. CPU - can this group use. For clarity, the above file-tree only shows these files at the deepest level.

Finally, here at leaves of tree are the files that describe how much memory (`memory.min` and `memory.max`), CPU (`cpu.weight` and `cpu.max`) or other resources each container gets to work with. These files are a direct translation of the resource requests and limits defined in Pod manifests. However, if you were to look at these files, the values you would find there actually don't seem related to the requests and limits, so what do they mean and how did they get there?

## How Does It Work?

Let's now walk through all the steps to better understand how the Pod `requests` and `limits` get translated/propagated all the way to files in `/sys/fs/...`.

We begin with simple Pod definition that includes memory and CPU requests/limits:

```

apiVersion: v1
kind: Pod
metadata:
  labels:
    run: webserver
  name: webserver
spec:
  containers:
  - image: nginx
    name: webserver
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

When we create/apply this Pod manifest, the Pod gets assigned to a Node and the `kubelet` on the Node takes this *PodSpec* and passes it to *Container Runtime Interface* (CRI), e.g. `containerd` or CRI-O, which translates it to lower level [OCI JSON spec](https://github.com/opencontainers/runtime-spec/blob/main/config.md#configuration-schema-example) that describes the container that will be created:

```
{
  "status": {
    "id": "d94159cf8228addd7a29beaa3f59794799e0f3f65e856af2cb6389704772ffee",
    "metadata": {
      "name": "webserver"
    },
    "image": {
      "image": "docker.io/library/nginx:latest"
    },
    "imageRef": "docker.io/library/nginx@sha256:ab589a3...c332a5f0a8b5e59db"
  },
  "info": {
    "runtimeSpec": {
      "hostname": "webserver",
      "linux": {
        "resources": {
          "memory": {
            "limit": 134217728,
            "swap": 134217728
          },
          "cpu": {
            "shares": 256,
            "quota": 50000,
            "period": 100000
          },
          "pids": { "limit": 0 },
          "hugepageLimits": [{ "pageSize": "2MB", "limit": 0 }],
          "unified":{
            "memory.high": "107374182",
            "memory.min": "67108864"
          }
        },
        "cgroupsPath":
          "kubepods-burstable-pod6910effd_ea14_4f76_a7de_53c333338acb.slice:crio:d94159cf8228addd7a29b...389704772ffee"
      }}}}
```

As you can see from the above, this spec includes `cgroupsPath` which is the directory where the cgroup files will be located. It also includes the already translated requests and limits under the `info.runtimeSpec.linux.resources` (we will talk about what these means a bit later).

This spec is then passed to lower-level OCI container runtime - most likely `runc` - which talks to *systemd* driver which creates systemd scope unit and also sets values in the files in `cgroupfs`.

To first inspect the systemd scope unit:

```

crictl ps
CONTAINER      IMAGE  CREATED        STATE    NAME       ATTEMPT  POD ID         POD
029d006435420  ...    6 minutes ago  Running  webserver  0        72d13807f0ab1  webserver


systemd-cgls --unit kubepods.slice --no-pager
Unit kubepods.slice (/kubepods.slice):
├─kubepods-burstable.slice
│ ├─kubepods-burstable-pod6910effd_ea14_4f76_a7de_53c333338acb.slice
│ │ └─crio-029d0064354201e077d8155f2147907dfe8f18ef2ccead607273d487971df7e0.scope ...
│ │   ├─6166 nginx: master process nginx -g daemon off;
│ │   ├─6212 nginx: worker process
│ │   └─6213 nginx: worker process
│ ├─kubepods-burstable-pod3fee6bda_0ed8_40fa_95c8_deb824f6de93.slice
│ └─ ...
│   └─...
│     └─...
└─...
  └─...
    └─...
      └─...

systemctl show --no-pager crio-029d0064354201e077d8155f2147907dfe8f18ef2ccead607273d487971df7e0.scope
MemoryMin=0
MemoryMin=67108864
MemoryHigh=107374182
MemoryMax=134217728
MemorySwapMax=infinity
MemoryLimit=infinity
CPUWeight=10
CPUQuotaPerSecUSec=500ms
CPUQuotaPeriodUSec=100ms
...
```

We first find the container ID using `crictl ps`, which is CRI equivalent of `docker ps`. In the output of this command we see our Pod `webserver` and the container ID. We then use `systemd-cgls` which recursively shows control groups content. In its output we see the group with our container's ID, which is `crio-029d006435420...`. Finally, we use `systemctl show --no-pager crio-029d006435420...` which gives us the systemd properties which were used to set the values in cgroup files.

To then inspect the cgroups filesystem itself:

```
cd /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/
ls -l kubepods-burstable-pod6910effd_ea14_4f76_a7de_53c333338acb.slice
...
-rw-r--r-- 1 root root 0 Dec  3 11:46 cpu.max
-rw-r--r-- 1 root root 0 Dec  3 11:46 cpu.weight
-r--r--r-- 1 root root 0 Dec  3 11:46 hugetlb.2MB.current
-rw-r--r-- 1 root root 0 Dec  3 11:46 hugetlb.2MB.max
-rw-r--r-- 1 root root 0 Dec  3 11:46 io.max
-rw-r--r-- 1 root root 0 Dec  3 11:46 io.weight
-rw-r--r-- 1 root root 0 Dec  3 11:46 memory.high
-rw-r--r-- 1 root root 0 Dec  3 11:46 memory.low
-rw-r--r-- 1 root root 0 Dec  3 11:46 memory.max
-rw-r--r-- 1 root root 0 Dec  3 11:46 memory.min
...

cat .../cpu.weight  


cat .../cpu.max  

cat .../memory.min  

cat .../memory.max  
```

We go to the directory `kubepods-burstable-pod6910effd_ea14_4f76_a7de_53c333338acb.slice` which was listed in the output of `systemd-cgls`. This is the cgroup directory for the whole `webserver` Pod. Here we find the individual cgroup files. The files and the values that we mostly care about are `cpu.weight`, `cpu.max`, `memory.min` and `memory.max` as these are the ones that describe CPU and memory requests/limits of the Pod, but what do the values mean?

-   `cpu.weight` - This is the CPU request. It is converted to the so-called *weight* (also called *"shares"*). It's in range 1 - 10000 and describes how much CPU will the container get in comparison to other containers. If you had only 2 processes on the system and one would have 2000 and the other 8000, then the former would get 20% and the latter 80% of CPU cycles. In this case `250m` equals to `10` "shares", so if we were to run Pod with `450m` CPU request, then it would get `18` "shares".
-   `cpu.max` - This is the CPU limit. The values in the file indicate that the group may consume up to `$MAX` in each `$PERIOD` duration. `max` for `$MAX` indicates no limit. In this case: consume `50000 / 100000`, therefore at most `0.5` (`500m`) CPU.
-   `memory.min` - This is memory request in bytes. This is only set if *memory QoS* is enabled in the cluster (explained later)
.-   `memory.max` - A memory usage hard limit in bytes. If a cgroup's memory usage reaches this limit and can't be reduced, the OOM killer is invoked in the cgroup and the container gets killed.

There's a lot of other files besides these 4, however none of them can be currently set through Pod manifests.

As a side note, if you want to poke around yourself, an alternative/faster way to find these values might be to get the path to the `cgroupfs` from container runtime spec mentioned earlier:

```
POD_ID="$(crictl pods --name webserver -q)"
crictl inspectp -o=json $POD_ID | jq .info.runtimeSpec.linux.cgroupsPath -r



ls -l /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod6910effd_..._53c333338acb.slice/

-rw-r--r-- 1 root root 0 Dec  3 11:46 cpu.max
-rw-r--r-- 1 root root 0 Dec  3 11:46 cpu.weight
-rw-r--r-- 1 root root 0 Dec  3 11:46 memory.high
-rw-r--r-- 1 root root 0 Dec  3 11:46 memory.low
-rw-r--r-- 1 root root 0 Dec  3 11:46 memory.max
-rw-r--r-- 1 root root 0 Dec  3 11:46 memory.min
...
```

## Monitoring

Apart from enforcing the resource allocation, cgroups are also used for monitoring resource consumption. This is done by *cAdvisor* component included in `kubelet`. Looking at the cAdvisor metrics also serves as an easier way to view the cgroups files values.

To view cAdvisor metrics you can use:

```

curl -sk -X GET  "https://localhost:10250/metrics/cadvisor" \
  --key /etc/kubernetes/pki/apiserver-kubelet-client.key \
  --cacert /etc/kubernetes/pki/ca.crt \
  --cert /etc/kubernetes/pki/apiserver-kubelet-client.crt


kubectl proxy &


kubectl get nodes
NAME        STATUS   ROLES           AGE   VERSION
some-node   Ready    control-plane   7d    v1.25.4

curl http://localhost:8001/api/v1/nodes/some-node/proxy/metrics/cadvisor
```

If you have access to the cluster Node, then you can get the metrics directly from kubelet API using the first `curl` command above. Alternatively, you can use `kubectl proxy` to get access to the Kubernetes API Server and run `curl` from local specifying one of your nodes in the path.

Regardless of which option you use, you will get a huge list of metrics which will look like [this sample](https://github.com/google/cadvisor/blob/master/metrics/testdata/prometheus_metrics).

Some of the more interesting metrics you will find there are:

```

container_spec_memory_limit_bytes{
  container="webserver",
  id="/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod691...38acb.slice/crio-d94159cf...04772ffee.scope",
  image="docker.io/library/nginx:latest",
  name="k8s_webserver_webserver_default_6910effd-ea14-4f76-a7de-53c333338acb_1",
  namespace="default",
  pod="webserver"} 1.34217728e+08




container_spec_cpu_quota{...} 50000



container_spec_cpu_period{...} 100000



container_spec_cpu_shares{...} 237
```

And as a final summary of the whole propagation and translation from Pod manifest all the way to `cgroupfs`, here's a little table:

Pod Spec

Systemd

Cgroups FS

Cadvisor Metric

`requests.memory`

`MemoryMin`

`memory.max`

`container_spec_memory_limit_bytes`

`limits.memory`

`MemoryMax`

`memory.max`

`container_spec_memory_limit_bytes`

`requests.cpu`

`CPUWeight`

`cpu.weight`

`container_spec_cpu_shares`

`limits.cpu`

`CPUQuotaPerSecUSec`

`cpu.max`

`container_spec_cpu_quota` (`_period`)

## Why Should You Care?

With all the newly acquired knowledge about cgroups in Kubernetes, there's a question, *"Why even bother learning this, when Linux and Kubernetes does all the work for us?"*. Well, deeper understanding is always beneficial in my opinion, and you never know when you will need this knowledge for debugging. More importantly though, knowing how it works, makes it possible to implement and take advantage of some advanced features:

-   For example *Memory QoS*, which was briefly mentioned earlier. Most people don't know this, but currently - as of Kubernetes v1.26 - memory requests in Pod manifest are not taken into consideration by container runtime and are therefore effectively ignored. Additionally, there's no way to throttle memory usage and when container reaches memory limit, it simply gets OOM killed. With introduction of [Memory QoS](https://kubernetes.io/blog/2021/11/26/qos-memory-resources/) feature, which is currently in *Alpha*, Kubernetes can take advantage of additional cgroup files `memory.min` and `memory.high` to throttle a container instead of straight-up killing it. (Note: The `memory.min` value in the earlier examples is populated only because Memory QoS was enabled on the cluster.)
-   Another possible advanced application for cgroups could be a *[container-aware OOM killer](https://www.scrivano.org/posts/2020-08-14-oom-group/)* - let's say you have a Pod with logging sidecar, if the Pod reaches memory usage limit, then the main container in the Pod might get killed because of memory consumption of the sidecar. With container-aware OOM killer, we could in theory configure the Pod so that the sidecar is killed first when memory limit is reached.
-   Thanks to cgroups, it's also possible to run Kubernetes components, such as `kubelet` or CRI in rootless mode ([using this Alpha feature](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-in-userns/)), which is great for security.
-   And finally, It's also good to have some knowledge of cgroups if you're a Java developer, because [JDK looks at the cgroup files](https://bugs.openjdk.org/browse/JDK-8230305) to understand how much CPU and memory is available.

And these are just the tip of the iceberg. There are many more things that cgroups can help us with and chances are, that in the future we will see more resource management features added to Kubernetes, for example for managing other types of resources such disk throttling, network I/O or [resource pressure (PSI)](https://docs.kernel.org/accounting/psi.html).

## Closing Thoughts

While a lot of the things in Kubernetes might seem like a magic, when you look closely you will find out that it's really just a clever use of core Linux components and features, and resource management is no exception as we've seen in this article.

Additionally, while cgroups might seem like an implementation detail from point of view of a cluster operator and user, having understanding of how they work can be beneficial when troubleshooting difficult issues or when using advanced features like the ones described in previous section.