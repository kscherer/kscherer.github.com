---
layout: post
title: "Benchmarking docker storage backends"
category: Docker
tags: [linux, docker]
---

I am using docker simulate building Wind River Linux (which is based
on OE-Core and Poky) on different hosts. The actual build is done on a
bind mount outside of the container so I did not expect the storage
backend to affect performance, but it did.

See https://github.com/docker/docker/issues/2891 for full history.

### Setup

- docker 1.7
- Ubuntu 14.04.2
- Vivid kernel 3.19.0-21-generic
- Dual 6C Xeon with 64GB RAM and 100GB root SSD and dual 3TB RAID0

Using following Dockerfile:

    FROM ubuntu:14.04.2

    MAINTAINER Konrad Scherer <Konrad.Scherer@windriver.com>

    RUN useradd --home-dir /home/wrlbuild --uid 1000 --gid 100 --shell /bin/bash wrlbuild && \
        echo "wrlbuild ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers

    RUN dpkg --add-architecture i386 && \
        apt-get update && \
        DEBIAN_FRONTEND=noninteractive apt-get -qy install --no-install-recommends \
        libc6:i386 libc6-dev-i386 libncurses5:i386 texi2html chrpath \
        diffstat subversion libgl1-mesa-dev libglu1-mesa-dev libsdl1.2-dev \
        texinfo gawk gcc gcc-multilib help2man g++ git-core python-gtk2 bash \
        diffutils xz-utils make file screen sudo wget time patch && \
        apt-get clean && \
        rm -rf /var/lib/apt/lists/* && \
        rm -rf /usr/share/man && \
        rm -rf /usr/share/doc && \
        rm -rf /usr/share/grub2 && \
        rm -rf /usr/share/texmf/fonts && \
        rm -rf /usr/share/texmf/doc

    USER wrlbuild

    CMD ["/bin/bash"]

Building poky, fido release, core-image-minimal on a ext4 bind mount with the docker
image using different storage backends.

    cd <buildarea>
    mkdir downloads
    git clone --branch fido git://git.yoctoproject.org/poky
    source poky/oe-init-build-env mybuild
    ln -s ../downloads .
    sed -i 's/#MACHINE ?= "qemux86-64"/MACHINE ?= "qemux86-64"/' conf/local.conf
    bitbake -c fetchall core-image-minimal
    time bitbake core-image-minimal

### Results

Bare-metal:

    real    33m5.260s
    user    289m41.356s
    sys     27m23.488s

Aufs:

    real    40m24.416s
    user    258m48.932s
    sys     56m29.284s

Devicemapper with official binary in loopback mode:
This requires `--storage-opt dm.override_udev_sync_check=true`

    real    35m24.415s
    user    289m10.660s
    sys     34m21.168s

Devicemapper with my own compiled dynamic binary:
This still requires `--storage-opt dm.override_udev_sync_check=true`
even though docker info states udev sync is supported.

    real    34m18.387s
    user    294m1.720s
    sys     31m43.764s

Overlayfs:

    real    33m46.890s
    user    293m40.084s
    sys     35m31.480s

### Conclusion

Aufs still has a measurable performance overhead even when the IO is
done on a bind mount outside of the aufs filesystem. Devicemapper and
overlayfs do not add overhead to this specific scenario. I did have
problems with devicemapper on Ubuntu 14.04 and the 3.13 kernel, but
since I upgraded to the 3.16 kernel I have not had any problems with
devicemapper errors. The only problems I have add were related to the
udev sync detection and new requirement with Docker 1.7.

My options are:

- Ignore the udev sync requirement with a flag
- Compile and distribute my own dynamically linked version of docker
  and hope that docker will provide an official version on Ubuntu
- Switch to Overlayfs

There are reports of problems with Overlayfs and using rpm inside a
container. I will do some more testing with Overlayfs, but it seems my
best option now is to move all my Ubuntu builders to use Overlayfs.
