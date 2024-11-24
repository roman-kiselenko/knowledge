---
title: Is Your Go Application Really Using the Correct Number of CPU Cores?
source: https://nemre.medium.com/is-your-go-application-really-using-the-correct-number-of-cpu-cores-20915d2b6ccb
clipped: 2023-10-18
published: 
category: development
tags:
  - golang
read: true
---

![[Raw/Media/Resources/172be62a9ecfb76d5a6452e8a94bca58_MD5.jpg]]

In the world of modern cloud-native languages, Go has gained significant popularity for its efficiency and concurrency support through goroutines. However, when it comes to utilizing the appropriate number of CPU cores, Go applications face some challenges. This article explores the underlying issues and sheds light on why Go applications may fail to leverage the correct number of CPU cores, particularly in containerized environments like Kubernetes.

## Go’s GMP Scheduler

Go employs a unique scheduling model called GMP, where G represents goroutines, M represents kernel-level threads, and P represents available CPU cores. Multiple goroutines can share a single thread (M), which needs to be bound to a specific P during execution. If the number of CPUs perceived by the operating system differs from the number of available CPUs perceived by a Go process, even binding an M to a specific P may not guarantee its execution. Therefore, the ability to obtain an accurate count of available CPU cores directly affects Go’s scheduling efficiency.

![[Raw/Media/Resources/8c70684c4e4114a96ac25e5ea30daae4_MD5.png]]

## Resource Limits in Kubernetes

Suppose a user sets resource limits for a Go application in a Kubernetes environment using the following specification:

```yaml
spec:  
  containers:  
  - name: "my-go-app"  
    resources:  
      limits:  
        cpu: "4"
```

One might assume that Go, being a popular language in the cloud-native era, would have built-in support for Kubernetes and automatically recognize these limits. However, that assumption is not entirely accurate. Let’s conduct a small experiment:

If the `GOMAXPROCS` environment variable is not explicitly set when starting the Go process, it defaults to the output of the `runtime.NumCPU()` function, which represents the number of available CPU cores (i.e., the value of P). However, adding a print statement for NumCPU reveals that it actually outputs the number of CPUs on the node, regardless of the `limits.cpu` value.

```go
package main

import "runtime"

func main() {  
  print(runtime.NumCPU())   
}
```

## Limitations of runtime.NumCPU()

On Linux, the `runtime.NumCPU()` function relies on the `sched_getaffinity` system call, which only considers CPU core bindings and doesn’t take into account CPU resource restrictions imposed by cgroups in container environments.

For instance, in Docker, the `--cpuset-cpus` flag defines the CPU core numbers available for a container at runtime, while restricting CPU resource usage primarily relies on `--cpus=`. Only the former (cpuset) can be recognized by `sched_getaffinity`.

For more information, refer to the Docker [documentation](https://docs.docker.com/config/containers/resource_constraints/#configure-the-default-cfs-scheduler) on resource constraints.

## Accurate Calculation of CPU Cores

Fortunately, there is a Go [library](https://nemre.medium.com/github.com/uber-go/automaxprocs), that accurately calculates `GOMAXPROCS` by reading the cgroup file system.

This library supports both cgroup v1 and v2. In cgroup v1, it reads the `cpu.cfs_quota_us` and `cpu.cfs_period_us` files and calculates their quotient. Typically, these files are located under `/sys/fs/cgroup/cpu/` (automaxprocs retrieves the mount information to determine the actual location). In cgroup v2, it reads the corresponding fields representing quota and period in the `cpu.max` file and calculates their quotient.

Since the division result may not be an integer, an additional step of rounding down is performed.

## Challenges Beyond Go

It’s important to note that Go is not the only language facing difficulties in recognizing the available CPU cores accurately within container environments. Nginx, Envoy, and other frameworks also struggle with recognizing cgroup configurations. However, it’s been reported that Rust and Java (OpenJDK implementation) have dedicated solutions to handle cgroup configurations.

If your application meets the following criteria: Performs compute-intensive tasks or has critical business logic executed within a fixed number of workers. Deployed in a containerized environment. It’s worth investigating whether the framework you are using properly handles cgroup configurations.

## Performance Impact

Let’s examine the benchmark results of the uber automaxprocs library to see the impact of setting the correct `GOMAXPROCS` configuration on performance.

Data measured from Uber’s internal load balancer with 200% CPU quota (i.e., 2 cores):

![[Raw/Media/Resources/405aa09ae8ddd30399c768e5e1000f89_MD5.png]]

When `GOMAXPROCS` is increased above the CPU quota, we see P50 decrease slightly, but see significant increases to P99. We also see that the total RPS handled also decreases.

## Conclusion

The mismatch between perceived and actual CPU core availability in container environments poses a challenge for Go applications. Although Go relies on the `runtime.NumCPU()` function, it doesn’t consider the CPU resource limitations imposed by groups.

Developers must be aware of this limitation and consider utilizing libraries like [automaxprocs](https://github.com/uber-go/automaxprocs) to accurately determine the optimal number of CPU cores for their Go applications. As the cloud-native landscape evolves, addressing these challenges will ensure efficient resource utilization and enhance the performance of Go applications in containerized environments.

*Good luck and happy coding!*