---
layout: post
title: "The Kubernetes batch job memory challenge"
category: 
tags: []
---

### Batch Jobs in Kubernetes

The design of Kubernetes has its origins at Google as a platform for
microservices. It does support batch jobs but they do not have the
same level of support as microservices. The only difference between a
Pod and Job is that a Job does not restart automatically. 

#### Pod Abstraction

The Pod abstraction assumes that the CPU and memory usage of the
processes running inside a Pod are predictable and fairly
constant. When a Pod's memory usage is larger than its allocation, the
assumption is that there is a bug or memory leak and the process
should be killed.

#### Best Effort resource allocation

For Pods the default resource allocation is "Best Effort". When no CPU
or memory limits are defined the Pod is considered low priority and
does not have any resource limits. During my initial explorations with
K8s I created 10 Best Effort Jobs that compiled software. K8s started
all the Pods on a single Node. The compile jobs quickly used up all
available memory on the system and K8s started killing Jobs at random.

This behavior makes sense when the Pods are stateless and part of
Deployments and con be easily moved to other nodes. Of course the
compile jobs can just be restarted but it is wasteful to throw away
the work in progress. In this case "Best Effort" doesn't really seem
appropriate for running batch jobs.

#### PodInterAntiAffinity

Using the Pod AnitAffinity with the nodename and/or labels, the K8s
scheduler will spread the Pods out over a set of Nodes. This way if
the nodes have enough resources "Best Effort" pods will have the best
chance of not consuming too many resources on the nodes. But it isn't
a guarantee and it is difficult to predict if a Job/Pod will finish.

#### Burst and Guaranteed Pods

If "Best Effort" isn't appropriate, then the next step is to give each
Job a CPU and memory limit. The question then becomes what those
limits should be? A Burstable Pod has lower limit and max limit. A
Guaranteed Pod has a max limit. A Burstable Pod allows a form of
resource over commitment. If the memory usage of the processes in the
Pod ever tries to allocate more memory than the resource limit, the
allocation will fail and K8s will kill the Pod.

#### Compressible and Uncompressible resources

The default Pod resources CPU and Memory have different behavior when
the resource is exhausted. CPU over commitment is handled by the
kernel scheduler and the scheduler will allocate CPU time based on the
scheduler policy. A Pod that attempts to use more CPU time than
allocated will be throttled. CPU is considered a compressible
resource.

Memory is different because if there isn't any memory available,
attempted allocations will fail. This is considered a uncompressible
resource. There isn't a way to throttle process memory usage. The only
option is swap memory which can allow allocations to proceed but it
comes with problems like thrashing. With containers it gets even more
complicated because by default the kernel does not account for swap
memory in the memory resource usage.

#### Setting memory limits

So the only way to prevent a Pod from being killed when it uses too
much memory is to set the memory allocation high enough. This is where
the predictability of microservices makes figuring out the max memory
limit easier.

#### Finding a memory limit for Yocto builds

But what about large and unpredictable Yocto builds? Yocto provides
infinite configuration options and also supports two forms of build
parallelization: jobs and parallel packages. These options speed up
the build but make the the package build order non
deterministic. SState can be used to speed up builds but it makes
predicting memory usage even more difficult because it is impossible
to know ahead of time which packages will be rebuilt.

### Swap?

What about using swap to add memory temporarily to a Pod that has used
all of its allocation? K8s does not support swap and will require that
swap be disabled. As far as I can tell the main issue is around how to
account for the swap memory. Swap cannot just be added to memory of
the machine because it is much slower. The current kube tools do not
track swap usage and the kernel does not enable swap memory accounting
by default for performance reasons. Swap can increase performance by
removing unused memory pages to disk and giving applications more
RAM. However a Pod using swap could thrash the entire node causing
failures of Pods that are not using too many resources.

There is work underway to add swap alpha support to K8s 1.22 but it
has many limitations and may be restricted to "Best Effort"
Pods. Exactly how swap will be managed and accounted for are still
open questions. At some point swap may be a part of the solution but
it isn't feasible now.

### Workarounds

Without a way to make memory compressible there are only workarounds
and tradeoffs.

- reducing parallelism for large packages like tensorflow or chromium
  does keep memory usage down at the cost of slowing down the
  build. When a build fails due to resource allocation failure we
  track the current packages being compiled from the process list and
  use that to identify potential problem packages.
- Tracking memory usage of a Pod using monitoring systems like
  Prometheus allows us to track changes in memory usage over time. It
  also gives us a better picture of how the memory usage of Yocto
  build changes over the course of the build. Ideally the processes
  causing peak memory usage can be identified and reduced to keep
  memory usage more stable allowing for better utilization of the
  nodes resources. The Prometheus alerting system can also be used to
  warn when builds cross memory usage boundaries like 90% which may
  give us time to deploy workarounds before builds fail.
- A build that failed due to a failed memory allocation could
  potentially be restarted with lower parallalization and/or changed
  memory limits. This is tricky because our builds use local disks
  with HostPath and the build Pod would need to be rescheduled to the
  node with the in progress build files. K8s 1.21 has added improved
  support for local volumes. Local volume management in K8s deserves a
  post of its own.
- The make tool has an option --load-average which tells make to no
  longer spawn new jobs if the load average is above a specific
  value. This doesn't work for Pods because load is system wide and
  uses CPU but concept of feedback into make or ninja is
  interesting. Since a process can use the /proc filesystem to monitor
  the memory usage of a container is might be possible to have these
  tools reduce the number of spawned jobs based on current container
  memory usage.

### Conclusion

It is the combination of the following:

- memory being uncompressible
- K8s/container memory limits are hard limits without second chances
- Yocto builds have unpredictable memory consumption

that makes this a very difficult problem. The only "proper" solution
would be to make the builds have more predictable memory usage. This
would require a feedback mechanism to make/ninja/bitbake to adjust the
number of running processes based on the current container memory
usage.
