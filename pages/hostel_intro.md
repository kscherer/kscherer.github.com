---
layout: page
title: "Host Test Lab Introduction"
---

# {{ page.title }}

## What is a Host Test Lab?

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

## Virtualization

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
collects IO requests and if this is true, then this could be the
bottleneck.

I plan to re-run this test to see if the KVM performance has
improved.

## Provisioning

The standard virtualization method uses images to create new
instances. With all the combinations of distro and architecture I was
looking at over 30 different images. I was also worried that the
images would not be easily moved from one virtualization solution to
another. I did some research and found
[Cobbler](https://github.com/cobbler/cobbler) which was the only
provisioning system to list support for RedHat, Debian _and_
Suse. Using cobbler to manage network or PXE installs simplifies that
creation of all the VMs. In the end all I had to manage was 5 files: a
kickstart for all RedHat and Fedora systems, 2 preseeds for Ubuntu and
Debian and 2 autoyast.xml files for OpenSuse and SLED.

Pros of cobbler approach
- Manage only templated kickstart files, no images
- No lock-in to virtualization solution. Also works with bare metal
- Excellent integration with DHCP and DNS for provisioned machines

Cons
- Extra time to learn kickstart, preseed and autoyast
- Can be difficult to fit into existing DHCP and network
  infrastructure
- Debian and OpenSuSE are second class citizens
- Puppet integration lacking but improving

I have written several other documents on setting up Cobbler on Debian
Squeeze and the other infrastructure needed to get all these distros
installed

[Cobbler Package Port from Ubuntu to Debian]({{ site.baseurl }}pages/cobbler_debian_port.html)
[Cobbler Setup]({{ site.baseurl }}pages/cobbler_setup.html)

I will be uploading the configs to their own github repo.

## Puppet

It is not enough to just install the machines and walk away. Software
configuration changes, hardware needs to be replaced, network
configuration changes, etc. That is where a configuration management
tool like Puppet comes in.

Puppet is several things:

- computer language for describing the state of various computer
resources like files and services
- a runtime environment to compare the current state of system against
  a description written in the Puppet language.
- an optional client-server application

To simplify resource definition, Puppet includes a tool called facter
which collects facts about the current state of a machine like
architecture, linux kernel version, etc. The facts are then made
available inside the Puppet language to make decisions about the
desired configuration. For example, the ntp configuration can have a
Debian and RedHat version.

There is also an abstraction layer for things like services and
packages. Puppet by default uses yum on RedHat machines and apt on
Debian machines. It cannot completely hide the package naming
differences but it can simplify them.

I also plan to upload the Puppet classes used at Wind River and some
of the configuration used to allow multiple users to collaborate.
