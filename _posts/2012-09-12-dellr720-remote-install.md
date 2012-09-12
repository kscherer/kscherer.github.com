---
layout: post
title: "Dell R720 Server install"
tags: []
---

Got a brand new Dell R720 server to install recently. The task sounded
simple enough: Install CentOS 6.3 x86_64 on the machine as quickly as
possible. The configuration of the server included 6 2TB drives which
will be used to store the code that will be shipped to the customer,
so RAID0 is not a good choice. A good place to start would be the RAID
configuration.

The server comes with iDRAC7 and it allows me to connect to the server
which was physically located over 3000km away. Linux and the iDRAC6
version of the VNC viewer did not work with arrow keys. A crazy hack
was necessary. It is described in detail here:
https://github.com/pjr/keycode-idrac. This was fixed with
iDRAC7. Progress!

Out of the box, Dell grouped the 6 disks into a RAID5 disk group, but
on top of that were 5 2TB and 1 80GB virtual disks on this disk
group. Further research shows that the BIOS cannot boot partitions
larger that 2TB and that many older file systems cannot handle disks
larger than 2TB. But newer technologies like
[GPT](http://en.wikipedia.org/wiki/GUID_Partition_Table) and
[ext4](http://en.wikipedia.org/wiki/Ext4) can handle theoretically
handle this. Let's give it a whirl.

In the RAID controller BIOS, the 6 virtual disks are deleted and one
massive 9TB disk is created. This disk will need to be booted using
UEFI.

Next step, go into the system setup. The BIOS now uses the latest
[UEFI](http://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)
and the arrow key that worked in the text mode bios now no longer
work. However after pressing the Num Lock key, the UEFI options can be
navigated using the keyboard. This is very annoying, but workable.

Under Boot Setting the boot system is switched to UEFI. To install
CentOS from CD, the virtual media option on the iDRAC uses a special
USB device to attaches the CentOS 6.3 install iso as CD on the
server. After waiting almost 5 minutes for the server to reboot, the
UEFI boot from Virtual CD fails. Oh well back to the old BIOS booting.

This means a smaller boot disk will be necessary. Back into the RAID
controller BIOS, delete the single disk and recreate one 500GB disk
and a larger 8.5TB disk. This time, the Virtual CD is found and the
install proceeds as usual.

But upon reboot, the system hangs and the system does not boot! I use
the CentOS netinstall CD as rescue to find out that the kickstart
decided to install into the large data drive, that the BIOS cannot
boot! To tell kickstart to ignore the large data drive it was necessary
to add:

    ignoredisk --drives=/dev/disk/by-path/pci-0000:03:00.0-scsi-0:2:1*

to the partition information in the kickstart and reinstall. This time
the install is successful and the server reboots into CentOS 6.3 and
puppet does the base configuration.

Next step is to prepare the large data drive to hold the data. One of
the irritations of using ext and many other filesystems is that fsck
is only possible when the disk is offline. For servers, this means
that fsck never gets run. Occasionally the server is rebooted and an
fsck is started. But this is usually the worst possible time to do
it. The result is that fsck is completely disabled and fingers are
crossed. One potential solution is e2croncheck.

It uses lvm read only snapshots to run fsck on a disk without making
it necessary to take the disk offline. There are a couple caveats of
course:
- There is a performance impact of running fsck on a disk
- The disk must obviously be on an lvm partition
- There must be free space in the volume group to hold any writes that
  are done while the snapshot is active

With this and alignment issues in mind, the following commands were
used to create the lvm and ext4 partition:

    parted -s /dev/sdb mklabel gpt
    parted -s /dev/sdb mkpart data ext2 1M 100%
    pvcreate --dataalignment=1M -M2 /dev/sdb1
    vgcreate vg /dev/sdb1
    lvcreate -L 8T -n git vg
    mkfs.ext4 -m 0 -E stride=16,stripe_width=80 /dev/mapper/vg-git

The RAID controller uses 64Kb block size, so I use a partition offset
and a data alignment of 1Mb to ensure all blocks line up on
boundaries. To help ext4 work within the RAID effectively,
stride=4K*16=64K and stripe_width=(6 disks - 1 parity = 5) * 16.

The server is now finally ready to be used.
