---
layout: page
title: "Host Test Lab Infrastructure"
---

# {{ page.title }}

## Mount ISO files as normal user


Cobbler supports cobbler import to simplify installation of various
Linux distributions. It scans a mounted iso for any relevant kernels,
etc and creates a distro and profile for that linux distribution. One
option is to loopback mount the iso files locally, but this requires a
lot of disk space and requires root access. In our office we also
already have a local mirror which already contains all the isos that I
need. Cobbler supports import of remote loopback mounted iso with the
--available-as parameter. To make this IT policy friendly, we can use
fuseiso to make it possible to normal user to loopback mount iso
files. What follows is a guide to combining all of these things.

## Setup fuseiso

On Ubuntu and Debian it is simple
    sudo aptitude install fuseiso
    
On Redhat EL5, the fuse-iso package is only available in rpmforge.

    sudo rpm -Uvh http://packages.sw.be/rpmforge-release/rpmforge-release-0.5.2-2.el5.rf.i386.rpm
    sudo rpm -Uhv http://dl.fedoraproject.org/pub/epel/5/i386/epel-release-5-4.noarch.rpm
    sudo yum install fuse-iso
    
On RedHat 5.1 I had to jump through a few more hoops because fuse did
not become part of offical RedHat repo until 2009.

    sudo rpm -Uvh http://elrepo.org/elrepo-release-5-3.el5.elrepo.noarch.rpm
    sudo rpm -Uvh http://dl.atrpms.net/all/atrpms-repo-5-5.el5.i386.rpm
    sudo yum install fuse-iso kmod-fuse
    sudo /sbin/modprobe fuse
    
By default fuse restricts access to the user that mounted the
file. The following lets non root users use the -oallow_others mount
option

    sudo echo "user_allow_other" > /etc/fuse.conf

Root access is no longer necessary.

## Mount ISO files

To mount an iso the command is:

    fuseiso image.iso /mount/path -oallow_other
    
To unmount the command is:

    fusermount -u /mount/path
    
To automate the process I use code like this:

    for version in 5.1 5.2 5.3 5.4 5.5 5.6 5.7 6.0 6.1 6.2
    do
       for arch in i386 x86_64
       do
           echo "Mount Redhat $version $arch"
           type=client
           if [ "${version:0:1}" == "6" ]; then
               type=workstation
           fi
           repodir=$repo_base/redhat-$version-$arch
           if [ ! -e $repodir ]; then
               mkdir -p $repodir
           fi
           mount | grep -q $repodir
           if [ $? -ne 0 ]; then
               fuseiso $redhat_iso_base/rhel-$type-$version-$arch-dvd.iso $repodir -oallow_other,ro
           fi
       done
    done

## Cobbler Distro Import

Cobbler supports "import" of distros which scans an existing directory
for valid kernels, etc and creates the corresponding distro and
profile. This import has many options including --available-as. This
options tells cobbler to ignore the current location of the files and
report them as available at a different http location. With the iso
files loopback mounted on another machine, this is very
useful. The idea is to have the kernel and initrd local to the cobbler
server and point the yum and deb repos on the mirror.

My first attempt was to nfs mount the files from the mirror to the
cobbler server, but nfs does not allow exports to cross filesystems
and the fuse mounted dirs are not exported.

My second attempt was to use the cobbler support for importing using
ssh and rsync. I had to setup rsync on the mirror. I modified
/etc/xinetd.d/rsync to enable the rsync daemon and created
/etc/rsyncd.conf.

    uid = nobody
    gid = nobody
    use chroot = no
    max connections = 4
    pid file = /var/run/rsyncd.pid

    [repos]
       path = /yow-lpggp14/mirror/repos
       read only = yes
       list = yes
       comment = Area with loopback mounted iso files

This works but copies the entire iso contents which negates the point
of having a mirror. It also messes up the available-as support.

The third attempt was to rsync the isos locally but to exclude the
following files using the modified file /etc/cobbler/rsync.exclude:

    **/debug/**
    **/alpha/**
    **/source/**
    **/SRPMS/**
    **/*.iso
    **/*.rpm
    **/*.deb
    *-repo/**
    **/kde-i18n**
    pool/**/*.dsc
    pool/**/*.gz
    
I created the dir /repos/local and ran the following:

    rsync -avlr --exclude-from=/etc/cobbler/rsync.exclude rsync://yow-lpggp1/repos/ .

This copies the CDs in only 13GB.

At this point I realized that cobbler import was really broken. So the
distro and profile need to be created manually.

