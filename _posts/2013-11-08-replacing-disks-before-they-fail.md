---
layout: post
title: "Replacing disks before they fail"
category: Linux
tags: [linux dell]
---

# Hardware setup

I am managing an R710 Dell server with 6 2TB disks. The RAID
controller does not support JBOD mode, so I had to create 6 RAID0
virtual disks with one disk per group. The disks are then passed
through to Linux as /dev/sda to /dev/sdf. I am running 6 xen vms and
each vm gets a dedicated disk. The vms are coverage builders and not
mission critical so there is no point in added redundancy. I have a
nice Cobbler/Foreman setup that makes provisioning very quick.

## OpenManage and check_openmanage

I am running the Dell OpenManage software on the system. If fact I am
running it on all my hardware. I am using the [puppet/dell][1] module
graciously shared on Github. The OpenManage package does many things
including CLI query access to all the hardware.

Then I stumbled across [check_openmanage][2] which is a Nagios check
which queries all the hardware and notifies Nagios if there are any
problems. I had already used the Puppet integration with Nagios to
setup a bunch of checks for ntp, disk and some other services. To make
things even easier, check_openmanage is in EPEL and Debian. It did not
take much time to add this check to the existing checks.

## Predicted Failure

So once everything was setup, I started getting warned about many
things that I was not aware of like firmware out of date and that some
hard drives were predicted to fail. The output of check_openmanage
looks like this:

    WARNING: Physical Disk 1:0:4 [Seagate ST32000444SS, 2.0TB] on ctrl 0 is Online, Failure Predicted

A reasonably painless call to Dell and a replacement disk is shipped.

## Disk replacement

When a disk fails it has a really nice blinking yellow light. To make
things clean, I wanted to shutdown and delete the correct vm before
changing the disk. How to figure out the correct vm to shutdown.

    > omreport storage pdisk controller=0 pdisk=1:0:4
    Physical Disk 1:0:4 on Controller PERC 6/i Integrated (Embedded)
    Controller PERC 6/i Integrated (Embedded)
    ID                              : 1:0:4
    Status                          : Non-Critical
    Name                            : Physical Disk 1:0:4
    State                           : Online
    Failure Predicted               : Yes

    > omreport storage pdisk controller=0 vdisk=5
    List of Physical Disks belonging to Virtual Disk 5
    Controller PERC 6/i Integrated (Embedded)
    ID                              : 1:0:4
    Status                          : Non-Critical
    Name                            : Physical Disk 1:0:4

Okay found the correct physical disk and the associated virtual disk.

    > omreport storage vdisk controller=0 vdisk=5
    Virtual Disk 5 on Controller PERC 6/i Integrated (Embedded)
    ID                            : 5
    Status                        : Ok
    Name                          : Virtual Disk 5
    State                         : Ready
    Device Name                   : /dev/sdf

Okay I know that this physical disk maps to the device /dev/sdf and I
initiated a shutdown of the vm that uses that disk.

The disk with predicted failure has a flashing amber light which makes
it easy to figure out which one to swap.

Once the swap is complete run the following command to recreate the vdisk.

    omconfig storage controller controller=0 action=createvdisk raid=r0 size=max pdisk=1:0:4

And /dev/sdf is available once again.

[1]: https://github.com/camptocamp/puppet-dell
[2]: http://folk.uio.no/trondham/software/check_openmanage.html
