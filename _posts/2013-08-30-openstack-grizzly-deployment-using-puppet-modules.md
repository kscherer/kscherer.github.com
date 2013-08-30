---
layout: post
title: "OpenStack Grizzly deployment using puppet modules"
category: Linux Openstack 
tags: [openstack linux]
---
## Openstack Grizzly 3 node cluster installation

There is a lot of infrastructure that I leveraged to do this
installation:
- Local ubuntu mirror
- Debian Preseed files to automate installation
- Dell iDRAC and faking netboot using virtual CDROM
- Puppet master with git branch to environment mapping
- Git subtrees to integrate OpenStack puppet modules
- An example hiera data file to handle configuration

## Local Ubuntu mirror

Having a local mirror makes installations much simpler because
packages download very quickly. The ideal setup uses netboot because
the mirror already contains the kernel and initrd and packages needed
to do the installation. I used:

    ubuntu/dists/precise/main/installer-amd64/current/images/netboot/ubuntu-installer/amd64/linux
    ubuntu/dists/precise/main/installer-amd64/current/images/netboot/ubuntu-installer/amd64/initrd.gz

To create the mirror I used the [ubumirror][1] scripts provided by
Canonical.

## Debian Preseed

I already have some experience using debian preseed files to automate
installation of Ubuntu and Debian. The documentation is spread out all
over the Internet. Most of the preseed is just sets the local mirror
and network setup. The OpenStack related options were the disk layout
and adding the Ubuntu Cloud Archive.

### Openstack Compute Node disk layout

The machines I am using were purchased before I even knew OpenStack
existed. They were used for Wind River Linux coverage builds and the
simplest configuration uses 2 900GB SAS drives in RAID0. The builds
require a lot of disk space and builds on SSD and in memory provided
only a small speedup versus the increase in cost.

My idea was to use LVM and allow cinder to use the remaining space to
create volumes for the vms. Here are the relevant preseed options to
handle the disk layout.

    d-i partman-auto/method string lvm
    d-i partman-auto/purge_lvm_from_device  boolean true
    d-i partman-auto-lvm/new_vg_name string cinder-volumes
    d-i partman-auto-lvm/guided_size string 500GB
    d-i partman-auto/choose_recipe select atomic

There are 3 kinds of storage in OpenStack: instance/ephemeral, block and
object.

- Object storage is handled by swift and not part of this
  installation.

- Block storage is done by default using iscsi and LVM
  logical volumes. Cinder looks for a LVM volume group called
  cinder-volumes and creates logical volumes there.

- Instance/Ephemeral storage by default goes into /var on the root
  filesystem. This is why I made the root filesystem 500GB. But this
  does not allow live migration because the root filesystem is not
  shared. If the vm was booted using block storage then the iscsi
  driver can handle the migration of vms. Another option is to mount
  /var on a shared nfs drive.

### Ubuntu Cloud Archive

I added the cloud and puppetlabs apt repos in the preseed to prevent
older versions of packages being installed.

    d-i apt-setup/local0/repository string \
        http://apt.puppetlabs.com/ precise main dependencies
    d-i apt-setup/local0/comment string Puppetlabs
    d-i apt-setup/local0/key string http://apt.puppetlabs.com/pubkey.gpg

    d-i apt-setup/local1/repository string \
        http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/grizzly main
    d-i apt-setup/local1/comment string Ubuntu Cloud Archive
    d-i apt-setup/local1/key string \
        http://ubuntu-cloud.archive.canonical.com/ubuntu/dists/precise-updates/grizzly/Release.gpg

    tasksel tasksel/first multiselect ubuntu-server
    d-i pkgsel/include string openssh-server ntp ruby libopenssl-ruby \
        vim-nox mcollective rubygems git puppet mcollective facter \
        ruby-stomp puppetlabs-release ubuntu-cloud-keyring

## Dell iDRAC and faking netboot using virtual CDROM 

Unfortunately I do not have DHCP, PXE and TFTP in this subnet to do
netboot provisioning. I am working on this with our IT department. So
for now I have to fake it.

I grab the mini.iso from the Ubuntu mirror

    ubuntu/dists/precise/main/installer-amd64/current/images/netboot/mini.iso

This contains the netboot kernel and initrd. I can then log into the
Dell iDRAC and start the remote console for the server. Using Virtual
Media redirection, I connect the mini.iso and boot the server. Press
F11 to get the boot menu and select Virtual CDROM.

But using this directly means I have to type everything into a tiny
console window. So I modified the isolinux.cfg to change the kernel
params to load the preseed automatically

Mount mini.iso locally and copy the contents to the hard drive

    sudo mount -o loop mini.iso /mnt/ubuntu/
    cp -r /mnt/ubuntu/ .
    chmod -R +w ubuntu

Here are the contents of the isolinux.cfg after editing:

    default preseed
    prompt 0
    timeout 0

    label preseed
        kernel linux
        append vga=788 initrd=initrd.gz locale=en_US auto url=<server>/my.preseed priority=critical interface=eth0 console-setup/ask_detect=false console-setup/layout=us --

Then make a new iso:

    mkisofs -o ubuntu-precise.iso -b isolinux.bin -c boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -R -J -v -T ubuntu/

Then the process is almost completely automated. Except that the
server cannot download the preseed until the networking is
configured. This info can be added to the kernel params, but then I
would have to edit each iso for each server. With RedHat kickstarts I
was able to add a script that mapped MAC address to IP and completely
automate this. But with preseeds I need to manually enter the network
info. The proper solution is a provisioner like Cobbler or Foreman.

## Puppet master with git branch to environment mapping

I have setup my puppet masters based on the [post][2] by Puppetlabs:

I like this setup a lot. All development happens on my desktop and I
have a consistent version controlled collection of all modules
available to my systems. I am using it give some colleagues that are
learning puppet a nice environment that won't mess up my systems.

But I have some custom in-house modules and I want to put the
OpenStack puppet modules in the same git branch beside them. The
existing tools like puppet module and puppet librarian, etc. do not
work in this use case. I want to be able to use git for these external
repos and be able to easily share any patches I make with
upstream. Enter git subtree.

## Git subtrees to integrate OpenStack puppet modules

Git subtree is part of the git package contrib files. Enabling it on
my system was simple:

    cd ~/bin
    cp /usr/share/doc/git/contrib/subtree/git-subtree.sh .
    chmod +x git-subtree.sh
    mv git-subtree.sh git-subtree

Now I can go to my modules directory and add in the OpenStack puppet
modules

    for arg in cinder glance horizon keystone nova; do \
        git subtree add --prefix=modules/$arg \
          --squash https://github.com/stackforge/puppet-$arg stable/grizzly;\
    done

There are some more supporting modules like inifile, rabbitmq, apt,
vcs, etc. Look in openstack/Puppetfile for the full list.

Next was to enable the modules on my machines. First the hiera data
needs to added for the network config. I was inspired by Chris Hodge's
[video][3] and [hiera data][4]

The gist has some minor issues. I posted a [revised version][5].

The last piece is to enable the modules on the nodes
    
    node 'controller' {
        include openstack::repo
        include openstack::controller
        include openstack::auth_file
        class { 'rabbitmq::repo::apt':
            before => Class['rabbitmq::server']
        }
    }
    node 'compute' {
        include openstack::repo
        include openstack::compute
    }

## Conclusion

Most of this infrastructure already existed or I had already done in
the past. I was able to reimage 3 machines and have a working grizzly
installation in about 3 hours.

Many thanks to all people who have contributed to Debian, Ubuntu, Puppet and
the OpenStack puppet modules.

[1]: https://launchpad.net/ubumirror
[2]: https://puppetlabs.com/blog/git-workflow-and-puppet-environments/
[3]: http://www.youtube.com/watch?v=owpi1WF9dws
[4]: https://gist.github.com/ody/5718115
[5]: https://gist.github.com/kscherer/6383077
