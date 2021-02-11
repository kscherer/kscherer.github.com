---
layout: post
title: "Update: Git Server option bigFileThreshold"
category: Git
tags: [git]
---

# Introduction

Many years ago (2014) I setup the my git servers with the git option
core.bigFileThreshold=100k. This reduced memory usage dramatically
because git stopped compressing already compressed files. I have used
this option for many years without apparent problems until one of my
colleagues alerted me that cloning an internal mirror of the Linux
kernel from my git server was transferring over 9GB of data! Cloning
the same repo from kernel.org transferred only approx 1.5GB.

# So many repack options

When I looked at the bare repo everything seemed normal. The repo had
been repacked properly less than a month ago thanks to
grokmirror. There was a single pack file with a bitmap and a single
pack file that was 9.1GB! I tried all the standard repack commands:

    > git repack -A -d -l -b

and when that didn't help:

    > git repack -A -d -l -b -F -f

But nothing changed. Then my colleague reported that rebuilding with
the above options did work on his machine and reduce the git repo
size.  This meant that there must be a local setting on the server
that was causing the problem. I looked at the local ~/.gitconfig and
saw the bigFileThreshold option I had set so long ago. So I did a
quick experiment with:

    > git -c core.bigFileThreshold=512m repack -A -d -F -f

and it did indeed reduce the bare git repo from 9.1GB to 1.9GB! It
seems that there are ~200K files in the Linux kernel repo that are
over 100k and when they are not compressed the size of the repository
grows a lot!

Curious how large the files in the kernel repo can become I did a
checkout of the mainline kernel and looked for files of 100Kb.

    > find . \( -path */.git/* \) -prune -o \( -type f -size +100k \) | wc -l
    914

Four of these files are even over 10MB!

# Solution

Once the problem has been clearly identified the solution is usually
simple. In this case the gitolite config for all the kernel repos sets
the core.bigFileThreshold to its default value of 512m. This way all
the other repos can still use the smaller bigFileThreshold setting.
