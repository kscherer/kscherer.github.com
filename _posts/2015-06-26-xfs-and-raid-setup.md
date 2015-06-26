---
layout: post
title: "XFS and RAID setup"
category: 
tags: [Linux]
---

## Choosing XFS

I manage a cluster of builder machines and all the builders use the
ext4 filesystem. To load the machines effectively, the builds are
heavily parallelized and using a RAID0 striped setup avoid IO
being a bottleneck on the builds. When RedHat 7 was released the
default filesystem was changed to xfs, I realized that it would be
a alternative to ext4 because RedHat wouldn't have made that change if
xfs wasn't a fast and solid filesystem. I recently got some new
hardware and started an experiment.

## Default RAID settings

The system has 6 4TB disks and I created 2 RAID0 disks of 3 disks each
for a total of 12TB per drive. The machine has a battery backed RAID
controller and each drive had as its default settings: stripe size of
64KB, write back, adaptive read ahead, disk cache enabled and a few
more.

## Creating the xfs drives

Once the machine was provisioned, I started reading about xfs
filesystem creation options and mount options. There were several
points of confusion:

- Some web pages referred to a crc option which validates
  metadata. This sounds like a good idea, but is not available with
  the xfsprogs version on Ubuntu 14.04

- I didn't realize at first that the inode64 option is a mount option
  and not a filesystem creation option

Since the disks are using hardware RAID which is not generally
detectable by the mkfs program, the geometry needs specified when
creating the drive.

    parted -s /dev/sdb mklabel gpt
    parted -s /dev/sdb mkpart build1 xfs 1M 100%
    mkfs.xfs -d su=64k,sw=3 /dev/sdb1

These commands create the partition and tell xfs the stripe size and
number of stripes.

## XFS mount options

It was clear that the inode64 was useful because the disks are large
and the metadata is spread out over the drive. The interesting option
was the barrier entry. There is an entry in the [XFS Wiki FAQ][1]
about this situation. If the storage is battery backed, then the
barrier is not necessary. Ideally the disk write cache is also
disabled to prevent data loss if the power is lost to the machine. So
I went back the RAID controller settings and disabled the disk cache
on all the drives and then added `nobarrier,inode64,defaults` to the
mount options for the drives.

## Conclusion

The experiment has started. The first build on the machine was very
fast, but the contribution of the filesystem is hard to determine. If
there are any interesting developments I will post updates.

[1]: http://xfs.org/index.php/XFS_FAQ#Q._Should_barriers_be_enabled_with_storage_which_has_a_persistent_write_cache.3F
