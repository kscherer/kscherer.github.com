---
layout: page
title: "VM installation using Foreman and Xen"
---

# {{ page.title }}

My experience with Cobbler has been good but several facts have made
me switch:

1. Development on Cobbler has essentially stopped after RedHat
   announced the future provisioner for Satellite would be Foreman
1. Cobbler never got a package for Debian
1. Puppet Dashboard was dropped by Puppetlabs

So I have moved to [Foreman][1] because it handles provisioning,
puppet reporting and is actively developed.

## Foreman Installation

The documentation for the installation is well done. The only problem
is that I cannot integrate the foreman puppet modules into my own due
to a name conflict with the puppet module.

I also am not a big fan of the webUI, but haven't had time to look
into the foreman CLI.

## VM setup in Foreman

The three levels in Cobbler made sense to me: distro, profile,
system. With foreman there are more levels to setup

1. Installation Media: Add a link to local mirrors and loop back
   mounted iso. Option to load xen kernel and initrd is missing. This
   is only relevant for old distros like RedHat/CentOS 5.x. Don't
   forget to set the distro family for the installation media
1. Partition Tables: The default ones are usable, but needed a tweak
   for RedHat 5.x based systems.
1. Provisioning templates: Setup kickstart and PXELinux
   templates. This is done in the WebUI which I don't really
   like. Luckily I didn't have to modify too much. The variable lookup
   is a pain, but I was able to make it work.
1. Operating System: Link the Media, Partition Table and Provisioning
   templates together. There is an annoyance that the OS cannot set
   the templates until the template is associated with the OS. But the
   template cannot be associated until the OS has been created.
1. Host: Finally a host can be associated with an OS.

Once that is all setup, the networking needs to setup and the host
added to the correct subnet. To verify that everything was setup
correctly, I had to look in /var/lib/dhcp/dhcpd.leases to see the dhcp
entry and /var/lib/tftpboot to make sure PXE template was correct.

I liked how easy it was to preview the kickstart/preseed templates.

## RedHat 5.x VM setup on Xen host

Cobbler had the Koan tool which completely automated the next
part. Back to the manual process for creating Xen PV guests. I tried
installing the guests as HVM with the PXE boot process, but on RedHat
5.10, the non-xen kernel got installed and the resulting image was
useless.

1. Retrieve the xen kernel and initrd manually and install on the local
   disk of the Xen host. I put them in /etc/xen/pxe.
1. Create a xm cfg file with the kernel and another with pygrub.
1. Boot the first config to install the system and then boot the
   second.

The pv pxe boot template:

    name = "vm"
    kernel = "/etc/xen/pxe/RedHat-5.10-i386-vmlinuz"
    ramdisk = "/etc/xen/pxe/RedHat-5.10-i386-initrd.img"
    extra = "ks=http://foreman/unattended/provision ksdevice=bootif network kssendmac"
    disk = [ "phy:<disk>,xvda,w" ]
    vif = [ 'mac=<mac>,bridge=<bridge>' ]
    vfb = [ 'type=vnc,vncunused=1,vnclisten=0.0.0.0' ]
    on_shutdown = 'destroy'
    on_reboot = 'destroy'

This will install the system and destroy itself when
finished. Otherwise, the vm will reboot, attempt reinstall and fail
when Foreman blocks access to the kickstart template

The pv boot template:

    name = "vm"
    bootloader = "/usr/lib/xen-4.1/bin/pygrub"
    disk = [ "phy:<disk>,xvda,w" ]
    vif = [ 'mac=<mac>,bridge=<bridge>' ]
    vfb = [ 'type=vnc,vncunused=1,vnclisten=0.0.0.0' ]

I didn't realize how much I had been spoiled with Koan. This is
doable, but takes some time. It can be automated, but I don't do it
often enough to justify fully automating the process.

## RedHat 6.x VM setup on Xen Host

The RedHat 6.x kernel works both in hvm and pv mode, so the hvm mode
can be used to install the rootfs. This avoids the coping step of the
kernel and initrd.

The hvm template:

    name = "vm"
    builder = 'hvm'
    kernel = "/usr/lib/xen-4.1/boot/hvmloader"
    boot = 'nc'
    disk = [ "phy:<disk>,xvda,w" ]
    vif = [ 'mac=<mac>,bridge=<bridge>' ]
    vfb = [ 'type=vnc,vncunused=1,vnclisten=0.0.0.0' ]
    on_shutdown = 'destroy'
    on_reboot = 'destroy'

The pv boot template is the same.

[1]: http://theforeman.org/
