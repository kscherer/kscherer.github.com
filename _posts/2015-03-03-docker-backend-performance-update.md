---
layout: post
title: "Docker backend performance update"
category: 
tags: []
---

A long time ago I filed Docker [2891][1] issue regarding the
performance of the aufs backend vs devicemapper.

Quick summary is that the aufs backend was approx 30% slower even
though the build was being done in a bind mount outside of the
container.

I finally got around to checking again using Docker 1.5 on Ubuntu
14.04 with 3.16 utopic LTS kernel.

The current stable poky release is dizzy:

    cd <buildarea>
    mkdir downloads
    chmod 777 downloads
    git clone --branch dizzy git://git.yoctoproject.org/poky
    source poky/oe-init-build-env mybuild
    ln -s ../downloads .
    bitbake -c fetchall core-image-minimal
    time bitbake core-image-minimal

There is no need to set the parallel packages and jobs now in
local.conf because bitbake now chooses reasonable defaults. 

Bare Metal:

    real    29m59.190s
    user    278m0.988s
    sys     59m47.379s

Devicemapper:

    real    32m21.074s
    user    281m53.994s
    sys     68m45.554s

AUFS:

    real    37m14.612s
    user    259m19.226s
    sys     85m50.269s

I only ran each build once so this is not an authoritative
benchmark. It shows that there is a performance overhead of approx 20%
when using the aufs backend even if the IO is done on a bind mount.

[1]: https://github.com/docker/docker/issues/2891
