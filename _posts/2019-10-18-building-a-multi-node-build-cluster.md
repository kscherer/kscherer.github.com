---
layout: post
title: "Building a multi node build cluster"
category: Build
tags: [linux]
---

## Introduction

I now have built, deployed and managed three internal build systems
that handle thousands (yes thousands) of Yocto builds daily. Each
build system has its tradeoffs and requirements. The latest one I call
[Wrigel][1] was specifically designed to be usable outside of WindRiver and
is available on our WindRiver-OpenSourceLabs GitHub repo and on the
Docker Hub. Recently there has been a lot of internal discussion about
build systems and the current state of various open source projects
and I will use this post to clarify my thinking.

## Wrigel Design Constraints

The primary use case of Wrigel was to make it easy for a team inside
or outside WindRiver to join 3-5 "spare" computers into a build
cluster. For this I used a combination of Docker, Docker Swarm and
Jenkins.

Docker makes it really easy to distribute preconfigured
Jenkins and build container images. Thanks to the generous support of
Docker Cloud all the container images required for Wrigel are built
and distributed on Docker Hub.

Docker Swarm makes is really easy to join 3-5 (Docker claims up to
thousands) systems together into a cluster. The best part is that
Docker Compose supports using the same yaml file to run services on a
single machine or distributed over a swarm. This has been ideal for
developing and testing the setup.

Jenkins is an incredible piece of software with an amazing community
that is used everywhere and has plugins for almost any
functionality. I rely heavily on the Pipeline plugin which provides a
sandboxed scripted pipeline DSL. This DSL support both single and
multi-node workflows. I have abused the groovy language support to do
some very complicated workflows.

I have a system that works and looks to be scalable. Of course the
system has limitations. It is these limitations and the current
landscape of alternatives that I have been investigating.

## Wrigel Limitations

Jenkins is a great tool, but the Pipeline plugin is very specific to
Jenkins. There isn't a single other tool that can run the Jenkins
Pipeline DSL. To be fair, every build tool from CircleCI to Azure
Pipelines and Tekton also have their own syntax and lock-in. There are
many kinds of lock-in and not all are bad. One of the perennial
challenges with all build systems has been reproducing the build
environment outside of the build system. Failures due to some special
build system state tend to make developers really unhappy, so I wanted
to explore what running a pipeline outside of a build system would
look like. I acknowledge the paradox of building a system to run
pipelines that also supports running pipelines outside of the system.

The other limitation is security. The constant stream of CVE reports
and fixes for Jenkins and its plugins is surprising. I am very
impressed with Cloudbees and the community with the way they are
taking these problems seriously. Cloudbees has made significant
progress improving the default Jenkins security settings. This is no
small feat considering Jenkins has a very old codebase. On the
downside my own attempts to secure the default setup have been broken
by Jenkins upgrades three times in the last year. While I understand
the churn I am reluctant to ship Jenkins as part of a potential
commercial product because each CVE would impose additional non
business value work on our team.

## Breaking down the problem

At its core, Jenkins is a cluster manager and a batch job
scheduler. It is also a plugin manager, but that isn't directly
relevant to this discussion. For a long time Jenkins was probably the
most common open source cluster manager. It is only recently with rise
of datacenter scale computers that more sophisticated cluster managers
have become available. In 2019 the major open source cluster managers
are Kubernetes, Nomad, Mesos + Marathon and Docker Swarm. Where
Jenkins is designed around batch jobs with an expected end time, newer
cluster managers are designed around the needs of a long lived
service. These managers have support for batch jobs, but it isn't the
primary abstraction. They also have many features that Jenkins does
not:

- Each job specifies its resource requirements. Jenkins only supports
  label selectors for choosing hosts
- The jobs are packed to maximize utilization of the systems. Jenkins
  by default will pack on a single machine and will prefer to reuse
  workareas.
- Each manager supports high availability configurations in the open
  source version whereas the HA feature for Jenkins is an Enterprise
  only feature
- Jobs can specify complex affinities and constraints on where the
  jobs can run.
- Each manager has integration with various container runtimes,
  storage and network plugins. Jenkins has integration with Docker but
  generally doesn't manage storage or network settings.

So by comparison Jenkins looks like a very limited scheduler, but it
does have pipeline support which none of the other projects does. So I
started exploring projects that add pipeline support to these
schedulers. I found many very new projects like Argo and Tekton for
Kubernetes, There are plugins for Jenkins that allow it to use
Kubernetes, Nomad or Mesos, but they can't really take advantage of
all the features.

## Cluster manager comparison

Now I will compare the features of the cluster managers which I feel
are most relevant to build cluster setup:

- How easy is the setup and maintenance?
- How complicated is the HA setup?
- Can it be run across multiple datacenters, i.e. Federated?
- Community and Industry support?

Docker Swarm:

- Very easy setup
- Automatic cert creation and rotation
- transparent overlay network setup
- HA easy to setup
- no WAN support
- Docker Inc. is focused on Kubernetes and future of Swarm is uncertain

Nomad:

- Install is a simple binary
- integration with Consul for HA
- encrypted communications
- no network setup
- plugins for job executors including Docker
- WAN setup supported by Consul
- Support for Service, Batch and System jobs
- Runs at large scale
- Well supported by Hashicorp and community
- job configuration in json or hcl

Mesos + Marathon:

- Support for Docker and custom containerizer
- No network setup by default
- Runs at large scale at Twitter
- Commercial support available
- Complicated installation and setup
- HA requires zookeeper setup
- no federation or WAN support
- Small community

Kubernetes:

- Very popular with lots of managed options
- Runs at large scale at many companies
- Supports build extensions like Tekton and Argo
- Federation support
- Lots of support options and great community
- Complicated setup and configuration
- Requires setup and management of etcd
- Requires setup and rotation of certs
- Requires network overlay setup using one of 10+ network plugins like Flannel

In my experience with Wrigel, Docker Swarm has worked well. It is only
its uncertain future that has encouraged me to look at Nomad.

## Running Pipelines outside Jenkins

Many years ago I saw a reference to a small tool on Github called
[Walter][6]. The idea is have a small go tool that can execute a
sequence of tasks as specified in a yaml file. It can execute steps
serially or in parallel. Each stage can have an unlimited number of
tasks and some cleanup tasks. Initially it supported only two stages
so I modified it to support unlimited stages. This tool can only
handle a single node pipeline, but that covers a lot of use cases. Now
the logic for building the pipeline is in the code that generates the
yaml file and not inside a Jenkinsfile. Ideally a developer could
download the yaml file and the walter binary and recreate the entire
build sequence on a local development machine. The temptation is to
have the yaml file call shell scripts, but by placing the full
commands in the yaml file with proper escaping each command could be
cut and pasted out of the yaml and run on a terminal.

## Workflow Support

It turns out that Jenkins Pipelines are an implementation of a much
larger concept called Workflow. Scientific computing has been building
multi-node cluster workflow engines for a long time. There is a list
of [awesome workflow engines][2] on Github. I find the concept of
directed acyclic graphs of workflow steps as mentioned by [Apache
Airflow][3] very interesting because it matches my mental model of
some of our larger build jobs.

With a package like [Luigi][4], the workflow can be encoded as a graph
of tasks and executed on a scheduler using "contribs", which are
interfaces to services outside of Luigi. There are [contribs][5] for
Kubernetes, AWS, ElasticSearch and more.

## Conclusion

With a single node pipeline written in yaml and executed by walter and
a multi node workflow built in Luigi, the build logic would be
independent of the cluster manager and scheduler. A developer could
run the workflows on a machine not managed by a cluster
manager. The build steps could be fairly easily executed on a cluster
managed by Jenkins, Nomad or Kubernetes.

[1]: https://github.com/WindRiver-OpenSourceLabs/ci-scripts
[2]: https://github.com/meirwah/awesome-workflow-engines
[3]: https://airflow.apache.org/
[4]: https://github.com/spotify/luigi
[5]: https://luigi.readthedocs.io/en/stable/api/luigi.contrib.html
[6]: https://github.com/walter-cd/walter
