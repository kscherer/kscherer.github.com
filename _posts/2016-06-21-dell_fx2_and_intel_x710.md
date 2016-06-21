---
layout: post
title: "Dell FX2 and Intel X710 nics"
category: Linux
tags: [linux, dell, fx2, x710]
---

What follows is an attempt to document a 6 month long debugging
odyssey. This is easily the strangest computer behavior I have ever
debugged or tried to understand.

### Background

I manage a cluster of bare metal servers used for coverage build
testing of Wind River Linux. The collection of git repos alone is over
15GB and the resulting IO traffic is high enough that using a public
cloud is not cost effective. The current sweet point for price to
build performance to rack space is a chassis that squeezes 4 blade
servers into a 2U chassis. We have a bunch of the Dell C6220 series
servers and then I decided to try the Dell FX2 chassis. The selling
points for me were the full Dell iDRAC and the network IO aggregator
system. The IO module theoretically would allow me to network 4 X
10GbE per system (160 GbE total) to a redundant switch pair providing
80 GbE uplink capability. We already had a good experience with the
M-8024K module for the M1000e chassis.

### Hardware setup

The first chassis was installed and networked. The IO modules were
setup in same way as the M-8024K which is as a VLAN access port. The
network was configured as VLAN 105, but with the access port this
detail is hidden from the systems. The main problem is that the RedHat
and Debian installers do not support VLAN configuration of the network
devices so all my machines have this configuration which allows me to
use Foreman for automated PXE installs.

### Using newest Ubuntu installer

Things were finally ready for me in January 2016. I started the PXE
install of Ubuntu 14.04. This failed because the kernel drivers for
the X710 nic were only integrated in Linux 4.2 and the Ubuntu
installer with the 3.13 kernel could not detect the X710 nic.

Luckily Ubuntu rebuilds the 14.04 installer image with the 15.10
kernel. I switched to the newest version of the installer and the
installer was able to detect the X710 nic.

### The first hiccup

This time the kernel and initrd were downloaded, the initial DHCP
succeeded but then DNS lookup to download the preseed failed. This was
strange but not unheard of. It had happened a long time ago but I
hadn't seen it in years. I was quick to blame our Microsoft DNS
servers and replaced all DNS names in the preseed with IP
addresses. This allowed the installation to complete and the machine
booted Ubuntu as usual. Then things started to get really
strange. DHCP on boot would occasionally fail and then I noticed that
DNS would occasionally time out and then succeed right
afterwards. This made using programs like Puppet impossible.

I completed the install of the other 3 servers and noticed that
occasionally that DHCP would fail during the install process. This was
mystifying to me because the PXE boot process uses the same DHCP setup
to download the kernel and initrd and I never saw it fail.

I checked the DHCP server and the server logs showed that the DHCP
received the request and was sending the offer back to the server, but
that offer was never received. Running ethtool did not show any
dropped or corrupted packets reported by the nic.

After the installation of the remaining 3 servers in the chassis was
complete, I opened a support case with Dell.

Approx two weeks:

TOR access port config, IOA VLAN 1 untagged, Hosts untagged = problem

### Round one - TOR switch config

The configuration of the TOR Cisco switch connecting to the IOA was
the subject of the first round of debugging. The IOA comes by default
in a "no touch" default configuration and it made sense to verify the
setup of the TOR switch. It took a few weeks to get together all the
people involved: myself, on site IT, IT networking, Dell tech support
and Dell networking specialist. After many hours, the TOR switch was
changed from an access port to a VLAN 105 tagged port. This resulted
in all traffic being dropped until the the IOA was changed to make the
105 VLAN untagged. But the DNS/DHCP problem persisted.

Round One - Approx one month

TOR VLAN 105, IOA VLAN 105 untagged, Hosts untagged = problem

### Round Two - Internal reproducer

Moving up levels of Dell support always takes time. While waiting for
networking support to become available I started experimenting. I
wanted to see if I could reproduce the problem without involving the
TOR switch so I setup dnsmasq on blade #1 as a dns caching proxy. I
then added a fake host entry into /etc/hosts so I could be sure that
dnsmasq was being queried and started running nslookup queries on
blade #2. To my surprise, I was able to reproduce the problem even
with the network traffic completely internal to the FX2 chassis.

Round Two - Approx one month

IOA VLAN 1 untagged, Hosts untagged = problem

### Round Three - A solution?

Then I decided to investigate if VLAN tagging at the Linux host level
would change things. I PXE booted blade #3 with the IOA configured
VLAN 105 untagged and when DHCP failed, I switched the IOA to VLAN 1
untagged and used the secondary install console to change the network
config from em1 to em1.105. I was able to complete the install and boot the
machine.

Amazingly the DHCP/DNS problems went away! It took some time to fix my
Puppet configuration to work with the VLAN tagging and get everything
working. I was also able to demonstrate that the problem using blade
#1 and #2 was present with VLAN 1 untagged and not present with VLAN 1
untagged and linux host configured for VLAN 105.

Round Three - Approx one month

IOA VLAN 1 untagged, Hosts tagged 105 = no problem!

### Round Four - Debugging the IOA

Now the focus turned exclusively to the IOA. With the help of Dell
network support, we disabled the outbound ports of the IOA and ran
tcpdump on the hosts. We were able to see packets being sent from the
DNS "server" and not being received by the client about 25% of the
time. About 5-10% of the time the initial DNS query would not even
make it to the DNS server.

It was around this time that a second FX2 chassis with identical
hardware arrived, but with a newer IOA firmware version. Full of hope
I did a PXE install on a blade, but with the exact same problem.

The Dell networking support team attempted to reproduce the problem on
their internal lab, but even with an FX2 chassis, Ubuntu 14.04 install
on an FC630 with the X710 nic they were unable to reproduce the
problem. To ensure the systems were configured identically we went
through the entire BIOS setup line by line to compare. I even tried
installs using UEFI and "Legacy BIOS" modes with no change in behavior.

I then got a crash course in F10 network configuration. It took a
while to find the proper command line incantations, but we setup
counters on the various ports to count incoming and outgoing
packets. We setup fixed ARP entries and tried to reduce the network
traffic as much as possible. Unfortunately the outgoing port counters
did not work, but from the incoming counters it looked like the IOA
was not seeing the packets come in the interface.

Round Four - Approx two months

IOA functioning as designed.

Bonus: I learned how to use the Dell iDRAC virtual media feature to
transfer files to and from a system without network access.

### Round Five - Debugging the X710 nic

Now another Dell Linux support tech was brought in and he confirmed
that the Linux config was correct. We then tried a firmware upgrade
for the X710 nic. This involved a failed upgrade attempt using an ISO
upgrade package (only works with Legacy bios mode), a DRAC upgrade with
HTML5 support and finally using the iDRAC upgrade functionality to
upgrade the NIC firmware.

Unfortunately the firmware upgrade made things even worse!! DHCP
worked but I could not ping inside the chassis. To make things even
more bizarre, ARP would occasionally work but ping would not!

At this point, we decided to replace the Intel X710 nic with the
Broadcom BCM57840 nic with a similar feature set to see how/if the
problem changed.

Round Five - Approx one month

Several failed firmware upgrades and violations of the laws of
networking.

### Round Six - Something goes right for a change

A technician swapped out the Intel Nics for Broadcom Nics. I redid
another PXE install (luckily it is completely automated) and
everything works as expected! No DHCP/DNS errors or any hint of
strange behavior.

We finally had a solution and the remaining Intel X710 nics were
swapped out over a few weeks.

Final setup:

TOR 105 access port, IOA VLAN 1 untagged default config, Linux host untagged.

### Recap

1. Bug not reproducible by support
1. Intermittent dropping of UDP packets without connection to non Dell
   hardware
1. Enabling VLAN tagging on the host "solved" the problem
1. Incorrect hardware counters
1. Firmware upgrades make things worse
1. Debugging requires coordination of at least 3 teams
1. Root cause never determined
1. Everyone involved agreed it was one of the strangest problems they
   have ever debugged

### Number of people involved

At Wind River: myself, IT and IT networking

At Dell: 2 tech support, 2 networking support, 1 Linux support, 2
managers

Total time consumed: approx 2-3 man months over 6 months of calendar
time.

### Conclusion

The reality is that Dell shipped us something broken. The open
question is whether testing could have found this problem before the
hardware shipped. Dell was unable to reproduce the problem internally
and without knowing the root cause of the problem, I can only
speculate.

Ideally I would like to know the cause of the problem, know that it
was fixed and that no one else will have to suffer through this. But
that would be the fairy tale ending and life doesn't work that
way. The case is considered closed and I will get back to all the
tasks I had to put on hold for this.
