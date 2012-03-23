---
layout: page
title: "Host Test Lab Introduction"
---

## {{ page.title }}

### What is a Host Test Lab?

The embedded Linux team at Wind River is a commercial Linux
distribution that most of our customers build from source. When
building _everything_ from source, there is a bootstrapping problem
where the build tools used to compile the source packages must first
be built using the tools installed on build computer. This makes the
build process very dependent on the state of Linux system the
distribution is being built on.

Ideally Wind River Linux will build on any Linux distribution that
our customer chooses, but realistically the supported distributions
has been restricted to:

* Redhat 4,5,6
* Fedora 13
* Ubuntu 10.04
* OpenSuSE 11.2
* SLED 10.2, 11.0

As newer versions are releases we also test build on them as time and
resources permits.

The Host Test Lab or Hostel is where all the coverage building on the
various Linux distributions is done.

### Virtualization

Running RedHat 4 on newer hardware is getting more and more difficult
and needing to be able to quickly create and erase installations of a
host distributions leads fairly obviously to virtualization. The first
test was to run a Wind River Linux build on the following
virtualization platforms:

* VMware
* Qemu and KVM
* VirtualBox
* Xen

This test was done in late 2009 on some Optiplex 760 (dual core
Core2Duo 4GB RAM) and the results were a little
surprising: VMware Player and Xen at 25% overhead, KVM at 100% and
VirtualBox at 200% overhead. 

I did some research but I can only speculate. Xen and VMware support
PCI passthrough where a PCI traffic is routed directly from guest to
physical hardware by the hypervisor. Qemu has an IO thread that
collects IO requests and this becomes the bottleneck.
