---
layout: post
title: "Starting Openstack deployment"
category: 
tags: [linux openstack packstack]
---

# Starting with Openstack

I have some experience with Xen, but no experience with any software
that controls a hypervisor. Wind River has several customers
interested in using oVirt and Openstack with Wind River Linux. Another
team is looking at oVirt, but no one had taken up the Openstack
investigation. I have experience with Puppet and Puppetlabs has some
official Openstack modules, so it seems a good place to start.

# Fedora 18

I re-purposed a coverage builder and installed Fedora 18. I had read
about Openstack and Fedora and thought that would be a good place to
start. Then Redhat announced the [Packstack][1] and [RDO][2] project
and I decided to give it a try.

The initial install failed due to selinux being disabled and conflicts
with NIS (our NIS deployment contains users with uids that conflict
with the ones in the rpms). When I finally got the packstack installer
to complete after a clean install, openstack refused to recognize the
admin user. So I did a reinstall using CentOS 6.4 and everything
worked without issue.

# Openstack and images

My experience with virtual machines has always been boot and install
onto some empty, usually virtual, disk. Openstack was my first
interaction with images. The docs recommend a base F18 image. The
first attempt to download using the horizon interface seemed to
hang. I was able to wget the image on my local machine 30 minutes
after the download using the web page had started.

# Openstack and LVM

My initial install of the host OS created a large LVM partition called
cinder-volumes for the Openstack Block storage service. Unfortunately,
the packstack installer renamed the volume group. I had to:

- Stop the cinder-volume service
- Delete the packstack created volume group and physical volume
- Rename the local LVM volume group
- Restart the cinder-volume service.

# Openstack and volumes

I went into the Volume section of the Horizon Web UI and created a
Volume. Running lvs on the host shows that a volume was created in the
correct place. I launched the instance as tiny and noticed that it did
not require a volume, which I found strange. I had setup a security
group to allow ssh and inject my public key. I then associated a
floating ip and was able to log into the vm! This was a happy moment.

After some poking around, a disk space check revealed that the VM had
10 GB disk space. This confused me because I had not associated it
with a volume. So I repeated the process but setup the VM to boot off
the volume I created earlier. This time the boot failed due to missing
boot image.

# Some EC2 history

I did more research and found this [article][3]. It explains some the
history of virtual machine infrastructure. When EC2 was first
launched, the VMs had no persistent storage. Customers had to use some
sort of web service like S3 to persist information. This kind of image
is called instance-store; Openstack refers to it as Ephemeral storage.

Then Amazon introduced EBS to provide persistent storage. It could be
attached to an instance-store image as a another block device. In
Openstack this is handled by Cinder as block level storage.

Then came the ability to boot from EBS volumes. This matches my
internal model of a virtual machine as persistent like a physical
machine. By default the volumes are empty, so the next step is
populating the volume with the proper bits. I have experience with
Cobbler to use kickstart and others to install new systems, but I was
curious if the image could be "transferred" to the volume.

The Horizon Web UI was not helpful. Some more research revealed the
following:

    cinder create --image-id <image-id> --display-name mybootable-vol 10

This runs qemu-img convert and writes the raw image to the new cinder
volume. This volume can be booted directly, but the Web UI still
requires an image name which is ignored.

# Summary

Types of Openstack VMs:
1) Ephemeral storage only. Default size of 0 means use image disk
size.
2) Ephemeral + block storage. VM must format if blank and
mount. Volume can only be attached to one VM.
3) Block storage only. Web UI does not support image to volume
conversion but cinder does.

Next on my list is NetApp cinder driver and installation on Ubuntu
12.04 Server using official puppet modules.

[1]: https://github.com/redhat-openstack/packstack
[2]: http://openstack.redhat.com/Main_Page
[3]: http://alestic.com/2012/01/ec2-ebs-boot-recommended
