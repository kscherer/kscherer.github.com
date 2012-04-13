---
layout: page
title: "Port Ubuntu cobbler package to Debian Squeeze"
---

# {{ page.title }}

Here is the scenario. I have a bunch of Debian Squeeze machines
running Xen. To create the VMs I am using PXE and network booting. I
am using cobbler to simplify this. I managed to manually install
cobbler myself on Debian Squeeze, but I noticed that Ubuntu has
created deb packages and I thought that I should be able to rebuild
those packages for Squeeze. To make things interesting I thought I
would do the rebuild on my Ubuntu 11.10 laptop which already has all
the build tools installed.

## Get Ubuntu packaging for Cobbler

Grab files from Launchpad for latest cobbler sources

    wget https://launchpad.net/ubuntu/+archive/primary/+files/cobbler_2.2.2.orig.tar.gz
    wget https://launchpad.net/ubuntu/+archive/primary/+files/cobbler_2.2.2-0ubuntu21.debian.tar.gz
    wget https://launchpad.net/ubuntu/+archive/primary/+files/cobbler_2.2.2-0ubuntu21.dsc
  
Unpack the files

    dpkg-source -x cobbler_2.2.2-0ubuntu21.dsc
  
Update the changelog

    EDITOR=emacsclient dch --newversion=2.2.2-1 --distribution=squeeze

Change debian/control file to change XS-Python-Version: >= 2.4
Remove cobbler.upstart file
Copy cobbler/config/cobblerd to cobbler/debian/cobbler.init

Setup pbuilder using instructions here:
https://wiki.ubuntu.com/PbuilderHowto

At the bottom section about adding local packages to the chroot, there
is a pbuilerrc which can handle both Ubuntu and Debian. Copy this to
~/.pbuilderrc and comment out the part with BINDMOUNT and OTHERMIRROR

Create squeeze environment using pbuilder

    sudo apt-get install debian-archive-keyring.gpg
    sudo DIST=squeeze pbuilder create --distribution squeeze --mirror http://yow-mirror.ottawa.wrs.com/mirror/debian --debootstrapopts --keyring=/usr/share/keyrings/debian-archive-keyring.gpg

Run pbuilder with squeeze environment

    DIST=squeeze pdebuild --debbuildopts -uc --debbuildopts -us
    
Check the dependencies

    dpkg-deb --info /var/cache/pbuilder/squeeze-amd64/cobbler_2.2.2-1_all.deb

Repeat with python-ethtool

## Make local debian repo

Easiest way to create local repo is with reprepro. Best intro into
setting up reprepro is here:
http://infrastructureanywhere.com/documentation/additional/mirrors.html#reprepro
