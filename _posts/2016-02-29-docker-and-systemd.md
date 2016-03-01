---
layout: post
title: "Docker Daemon and Systemd"
category: Docker
tags: [linux, docker, systemd]
---

I recently read an article on [LWN][1] about [Systemd vs Docker][2]
and I was disappointed. As far as I am concerned, this is preventing
one of the worst design flaws in Docker from being addressed. Docker
CEO Solomon Hykes also thinks this should be resolved, though
[Issue #2658][3] has remained open since Nov 2013.

The current Docker design sets up all containers as children of the
Docker daemon process. The consequence of this is that upgrading the
daemon requires that all the containers are stopped/killed. Other
operations like changing the daemon command line requires stopping all
the containers. I have to be extra careful with my Puppet
configuration because any change to the config files will restart the
docker daemon. To prevent inadvertent restarts I had to remove the
normal configuration to service dependency which normally restarts the
daemon when the configuration changes.

From an operational perspective this is a pain. It represents another
in a long line of software that requires significant operational
resources to deploy properly. If the operator is lucky, the
containerized application can be managed with load balancers or DNS
rotation. If the service cannot work this way or the Ops team cannot
build the required infrastructure, then upgrades mean downtime. With
VMs it is possible to move the application to another machine, but
CRIU isn't ready yet. These "solutions" require large amounts of
operational effort. I built a rolling upgrade system around Ansible to
handle docker upgrades.

My experience with [Mesos][4] has been very different. The Mesos team has a
[supported upgrade path][5] with lots of testing. I have upgraded at least
5 releases of Mesos without issues or any downtime.

What does this have to do with systemd? In order to support seamless
upgrades of the docker daemon, the ownership of the container
processes will have to be shared with some other process. This could
be another daemon, but the init system is an obvious choice. If the
docker daemon co-operated with another daemon or systemd by sharing
ownership of the processes, then a nice upgrade path could be
developed.

The Docker team is working on containerd and has stated that RunC
would be integrated and this may where better integration with an init
system becomes possible. I realize this is selfish, but for me all
these squabbles are just distracting developers from addressing one of
my major pain points with using Docker.

[1]: http://lwn.net/
[2]: http://lwn.net/Articles/676831/
[3]: https://github.com/docker/docker/issues/2658
[4]: http://mesos.apache.org/
[5]: http://mesos.apache.org/documentation/latest/upgrades/

