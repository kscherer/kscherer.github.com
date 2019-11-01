---
layout: post
title: "ZFS Disk replacement on Dell R730"
category: Linux
tags: [linux dell]
---

## Introduction

I manage a bunch of Dell servers and I use OpenManage and
check_openmanage to monitor for hardware failures. Recently one
machine started showing the following error:

    Logical Drive '/dev/sdh' [RAID-0, 3,725.50 GB] is Ready

Unfortunately "Drive is Ready" isn't a helpful error message. So I log
into the machine and check the disk:

    > omreport storage vdisk controller=0 vdisk=7
    Virtual Disk 7 on Controller PERC H730P Mini (Embedded)

    Controller PERC H730P Mini (Embedded)
    ID                                : 7
    Status                            : Critical
    Name                              : Virtual Disk 7
    State                             : Ready

The RAID controller log shows a more helpful message:

    Bad block medium error is detected at block 0x190018718 on Virtual Disk 7 on Integrated RAID Controller 1.

From experience I know that I could just clear the bad blocks, but the
drive is dying and more will come. Luckily Dell will replace drives
with uncorrectable errors and I received a replacement drive quickly.

## Cleanly removing the drive

I know the drive is /dev/sdh, but I created the ZFS pool using drive
paths. Searching `/dev/disk/by-path/` gave me the correct drive.

First step is to mark the drive as offline.

    > zpool offline pool 'pci-0000:03:00.0-scsi-0:2:7:0'

To make sure I replaced the correct drive I also forced it to blink:

    > omconfig storage vdisk controller=0 vdisk=7 action=blink

Next came the manual step of actually replacing the drive.

## Activating the new drive

After inserting the new disk I was able to determine the physical disk
number and recreate the RAID-0 virtual disk.

    > omconfig storage controller controller=0 action=createvdisk raid=r0 size=max pdisk=0:1:6

I use single drive RAID0 because I prefer that ZFS use the disks in
raidz2 mode rather than using RAID6 on the controller.

Then a quick verify that the new virtual disk is using the same PCI
device and drive letter and then add it back into the ZFS pool.

    > omreport storage vdisk controller=0 vdisk=7
    Virtual Disk 7 on Controller PERC H730P Mini (Embedded)

    Controller PERC H730P Mini (Embedded)
    ID                                : 7
    Status                            : Ok
    Name                              : Virtual Disk7
    State                             : Ready
    Device Name                       : /dev/sdh
    > parted -s /dev/sdh mklabel gpt
    > zpool replace pool 'pci-0000:03:00.0-scsi-0:2:7:0'

ZFS will add the new drive and resliver the data.

    > zpool status
    pool: pool
    state: DEGRADED
    status: One or more devices is currently being resilvered.  The pool will
            continue to function, possibly in a degraded state.
    action: Wait for the resilver to complete.

It would be slightly easier if the virtual disk was handled by the
RAID controller, but the rebuild would take much longer. So far ZFS on
Linux has worked very well for me and I will continue to rely on it.
