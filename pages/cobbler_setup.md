---
layout: page
title: "Setup cobbler on Debian Squeeze"
---

# {{ page.title }}

Install the necessary packages.

    aptitude install -R cobbler cobbler_web

Make sure service is started.

    service cobbler start

Some setting need to adjusted:
- createrepo_flags: "--pretty --cachedir cache --update --checkts --database --quiet"
- default_virt_bridge: br0
- default_virt_type: xenpv
- manage_dhcp: 1
- manage_dns: 1
- manage_tftp: 1

Change modules.conf to tell cobbler to use dnsmasq for dhcp and dns
and the internal tftp server.

    [dns]
    module = manage_dnsmasq
    [dhcp]
    module = manage_dnsmasq
    [tftpd]
    module = manage_tftpd_py

Note: to run the internal tftpd daemon, xinetd must be installed and started.

    aptitude install xinetd dnsmasq

The configuration for dnsmasq is controlled by cobbler. The file
/etc/dnsmasq.conf is overwritten by 'cobbler sync'. Modify the
/etc/cobbler/dnsmasq.template to contain the following (the dns server
and network params will need to change):
    read-ethers
    addn-hosts = /var/lib/cobbler/cobbler_hosts
    no-resolv #ignore /etc/resolv.conf
    server=128.224.144.58
    dhcp-range=128.224.194.20,128.224.194.254,255.255.255.0,128.224.194.255,24h
    dhcp-option=3,128.224.194.1
    dhcp-option=15,"wrs.com"
    dhcp-lease-max=1000
    dhcp-authoritative
    dhcp-boot=pxelinux.0
    dhcp-boot=net:normalarch,pxelinux.0
    dhcp-boot=net:ia64,$elilo
    #dnsmasq was returning 127.0.0.1 addresses from /etc/hosts
    localise-queries
    #puppet requires fqdn and this is necessary to make expanding unqualified
    #hostname to fully qualified hostname work
    expand-hosts

I also had to edit /etc/cobbler/tftpd.template to change $binary to
/usr/sbin/tftpd. I will report this bug in the Ubuntu package.

Restart cobbler and run cobbler sync to apply changes.

Add distros and profiles
----

Run cobbler_2.2_distro_import.sh script. This uses the base repos
synced (as mentioned in mountiso doc) and loads the pxe boot kernel
and initrd for all the supported distros.

The script runs cobbler reposync and cobbler sync to get the
filesystem up to date. 


Setup PXE boot for Debian
----

Copy linux and initrd.gz from Debian mirror to local system dir like /repos/debian-squeeze/
http://mirror/debian/dists/squeeze/main/installer-amd64/current/images/netboot/debian-installer/amd64/

Create the distro using the command line or the web UI

    cobbler distro add --kernel=/repos/debian-squeeze/linux
    --initrd=/repos/debian-squeeze/initrd.gz
    --kopts='console-keymaps-at/keymap=us' --breed=debian
    --os-version=squeeze --arch=x86_64 --owners=admin
    --name=debian-squeeze

Put xen-node.preseed in /var/lib/cobbler/kickstarts and create the profile

    cobbler profile add --name=debian-squeeze-xen
    --distro=debian-squeeze
    --kickstart=/var/lib/cobbler/kickstarts/xen-node.preseed

Note: I leave the kickstarts in a git repo that can be edited as
normal user and make a link into /var/lib/cobbler/kickstarts. This way
I do not have to be logged in as root and I can use git.

Next step is to create the system definition for the physical hardware.

    cobbler system add --name=yow-lpgbld-09
    --profile=debian-squeeze-xen --hostname=yow-lpgbld-09
    --dns-name=yow-lpgbld-09.wrs.com --ip-address=128.224.194.109
    --mac-address=00:24:E8:28:18:99 --netboot-enabled=true

To install on physical hardware, reboot the machine and select PXE
boot for the boot menu. The PXE boot should get an IP address over
DHCP and then download the kernel and initrd from TFTP and boot into
the installer.


Installation of RedHat VM
-----

Cobbler needs to know about the system, so use cobbler system add to
create the system. The mac, ip address, name etc will depend on the
network environment.

    cobbler system add --name=yow-lpgbld-vm9
    --profile=redhat-5.5-xen-i386 --hostname=yow-lpgbld-vm9
    --dns-name=yow-lpgbld-vm9.wrs.com --ip-address=128.224.194.29
    --mac-address=AA:00:00:00:00:09 --netboot-enabled=true
    
The profile links to redhat.ks which uses a lot of snippets to
work. Make sure all the snippets referenced in the kickstart are
copied or linked into /var/lib/cobbler/snippets. redhat.ks uses
snippets like base_install and post_install_script.

To install on virtual hardware, install koan on the Dom0. Koan stands
for "kickstart over network". The following command will retrieve the
configuration for the system registered in cobbler as vm9 and use
libvirt to kickstart the system.

    koan --server yow-lpgbld-master --virt
    --system=yow-lpgbld-vm9 --virt-name=yow-lpgbld-vm9 --virt-type=xenpv
    --virt-path=/dev/mapper/vg-vm --virt-bridge=br0 --vm-poll

