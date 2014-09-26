---
layout: post
title: "Git Server option bigFileThreshold"
category: Linux Git
tags: [linux git]
---

# Introduction

I manage the git infrastructure for the Linux group at Wind River: the
main git server and 5 regional mirrors which are mirrored using
grokmirror. I plan to do a post about our grokmirror setup. The main
git server holds over 500GB of bare git repos and over 600 of those
are mirrored. Many repos are not mirrored. Some repos are internal,
some are mirrors of external upstream repos and some are mirrors of
upstream repos with internal branches. The git server runs CentOS 5.10
and git 1.8.2 from EPEL.

# The toolchain binary repos

One of the largest repos contains the source for the toolchain _and_
all the binaries. Since the toolchain takes a long time to build, it
was decided that Wind River Linux should ship pre-compiled binaries
for the toolchain. There is also an option which allows our customers
to rebuild the toolchain if they have a reason to.

The bare toolchain repo size varies between 1 and 3GB depending on
supported architectures. Many of the files in the repo were tarballs
around 250MB size.

# Why is the git server down again?

When a new toolchain is ready for integration, it is uploaded to the
main git server and mirrored. Then the main tree is switched to enable
the new version of the toolchain and all the coverage builders start
to download the new version. Suddenly the git servers would become
unresponsive and would thrash under memory pressure until they would
be inevitably rebooted. Sometimes I would have to disable the coverage
builders and stage their activation to prevent a thundering herd from
knocking the git server over again.

# Why does cloning a repo require so much memory?

I finally decided to investigate this and found a reproducer
quickly. Cloning a 2.9GB bare repo would consume over 7GB of RAM
before the clone was complete. The graph of used memory was
spectacular. I started reading the git config man page and asking
google various questions.

I tried setting the binary attributes on various file types, but
nothing changed. See man gitattributes for more information. The
default set seem to be fine.

I tried various git config options like core.packedGitWindowSize and
core.packedGitLimit and core.compression as recommended in many blog
posts. But the memory spike was still the same.

# core.bigFileThreshold

From the git config man page:

    Files larger than this size are stored deflated, without attempting delta compression.
    Storing large files without delta compression avoids excessive memory usage, at the slight
    expense of increased disk usage.

    Default is 512 MiB on all platforms. This should be reasonable for most projects as source
    code and other text files can still be delta compressed, but larger binary media files
    won’t be.

The 512MB number is key. The reason the git server was using so much
memory is because it was doing delta compression on the binary
tarballs. This didn't make the files any smaller because they were
already compressed and required a lot of memory. I tried one command:

    git config --global --add core.bigFileThreshold 1

And suddenly (no git daemon restart necessary), the clone took a
fraction of the time and the memory spike was gone. The only downside
was that the repo required more disk space; about 4.5GB. I then tried:

    git config --global --add core.bigFileThreshold 100k

The resulted in approx 10% more disk space (3.3GB) and no memory spike
when cloning.

This setting seems very reasonable to me. The chance of having a text
file larger than 100Kb is very low and the only downside is slightly
higher disk usage. Git already is very efficient in this regard.
