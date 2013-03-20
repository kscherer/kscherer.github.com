---
layout: post
title: "Lessons learned running e2croncheck"
category: Linux
tags: [e2croncheck linux]
---

Filesystems (ext4, xfs, zfs, etc) are one of those things whose
failure nobody really wants to think about. The difference between a
hard disk failure and complete filesystem corruption is largely
academic. However a filesystem has many failure modes and the scariest
is silent corruption that goes undetected for a long time. Worst case
scenario is that backups are rendered useless.

The long time solution to detecting and correcting minor filesystem
issues is fsck. The tool has several limitations:
1. The check can only be run while the filesystem is offline.
2. The check is serial per filesystem. It can be parallelized across
multiple filesystems
3. As the amount of data on the filesystem grows, the time to complete
the check grows as well.

What seems to be standard practice is the following:
1. Install and configure system with defaults
2. Leave system running as long as possible
3. When the machine hangs at a critical moment, reboot the machine
4. Wait for hours until admin logs into console and fsck check is
manually killed

This has several obvious drawbacks:
1. fsck almost never gets a full run, especially if the system uses
hibernation and/or S3 sleep
2. The downtime always happens at the worst possible time
3. No one knows how long an fsck is actually going to take
4. The fsck may not be necessary, but the disk/machine needs to be
offline anyways

Online fsck seems to impossible, because the state of the filesystem
can change in ways that make the check wrong.

Databases have a similar problem: how to do a backup while the system
is in operation. The solution there is to use filesystem
snapshots. This is how I stumbled upon e2croncheck. The original from
Theodore Ts'o is [here][1]. I found a revised version on GitHub by
[Ion][2]

The script creates a read write snapshot of the filesystem. LVM uses a
copy on write snapshot volume to track changes to the original
filesystem. The script thens runs e2fsck on the snapshot which will
report if there is actual corruption on the filesystem that needs to
be repaired offline.

This seems like a better solution than the standard practice of
ignoring the problem so I setup my next servers in the following way:
* Six physical disks in hardware RAID 5
* Two virtual disks: 500GB system and 8.6TB data
* System uses ext4
* Data uses lvm with one lvm physical volume and one lvm volume group
* Single logical volume at 8TB with 500GB unused space for snapshot
* Cronjob to run e2croncheck weekly

LVM snapshots are not without their problems. The big one is
performance. There is overhead to the COW filesystem, but thanks to
the Internet I found some benchmarks comparing performance with
[chunksize][3]. The default chunksize is 4kB and increasing the
chunksize to 64kB increases performance by 10x!

I also added ionice with e2fsck set to idle priority. So far the
changes mean that the background check does not interfere with
programs that are running.

The final version of the script is located [here][4] inside a puppet
class to install the file and cron job.

[1]: http://ftp.sunet.se/pub/Linux/kernels/people/tytso/e2croncheck
[2]: https://github.com/ion1/e2croncheck/blob/master/e2croncheck
[3]: http://www.nikhef.nl/~dennisvd/lvmcrap.html
[4]: https://github.com/kscherer/puppet-modules/blob/production/modules/e2croncheck/files/e2croncheck
