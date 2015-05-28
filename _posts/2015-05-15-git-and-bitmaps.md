---
layout: post
title: "Adventures with Git and server packfile bitmaps"
category: Git
tags: [linux, git]
---

In git 2.0, a new feature called bitmaps was added. The entry from the
git [Changelog][1] has the following entry:

    The bitmap-index feature from JGit has been ported, which should
    significantly improve performance when serving objects from a
    repository that uses it.

One of my colleagues told me that he had experimented with it had
noticed some impressive speedups that I was able to reproduce. On the
local GigE network a linux kernel clone went from approx 3 minutes to
1.5 minutes, a speedup of almost 50%!

The instructions seemed very simple. Just log into the git server and
run:

    git repack -A -b

on every bare repo. The first hurdle was upgrading to a newer version
of git. Our git servers are running CentOS 5, CentOS 6 and Ubuntu
14.04. The EPEL version of git is 1.8 and 14.04 ships with 1.9.1.

For Ubuntu 14.04 the solution was to use the LaunchPad
[Git Stable PPA][2]

But for CentOS, it was a little trickier. Since I hate distributing
binaries directly I decided to backport the latest Fedora git
srpm. Getting it to build required a few hacks with bash completion
and installing a few dependencies, but it took less than 30 minutes to
get both CentOS 5 and 6 rpms.

The upgrade of git on the servers worked very well because they are
using xinetd to run the git-daemon and the very next connection to the
server after the upgrade started using the newly installed git 2.3.5
binary.

There were of course a few hiccups. An internal tool that used git
request-pull was relying on one of the working "heuristics" (see
changelog) that were removed.

The next step was to repack all the bare repos on the server. So I
wrote a script to run `git repack -A -b` and left it to run
overnight. Recovering from this the next few days would require me to
become very familiar with the git man pages.

First problem was that the git server ran out of disk space. Turns out
I needed to add the `-d` flag in order to delete the previous pack
files. I had effectively doubled the disk space requirements of every
repo!

It also turns out the `-A` leaves packfiles that contain dangling
objects. So I reran my script with

    git gc --aggressive
    git repack -a -d -b

This helped a lot but repos that were using alternates were still
taking a lot more space than before because repack was making one big
packfile of all the objects and effectively ignoring the alternates
file. This is documented in the `git clone` man page.

So I went to all the repos with alternates and ran:

    git repack -a -d -b -l

The `-l` flag only repacks files that are not available in the
alternates. With some extra cleanup, this resulted in even less disk
space usage than before. Unfortunately this does mean that a repo with
alternates cannot have a bitmap.

On one server many repos still did not contain the bitmap file. After
much experimentation I finally figured out that the pack.packSizeLimit
option had been set on the server only to 500M. This meant that repos
larger than 500M would have multiple pack files and since the bitmap
requires a single pack file, no bitmap was created. The lack of
warning extended the debugging time considerably.

Finally one of my servers had an old mirror of the upstream Linux
kernel repo and even after `git gc --aggressive` the repo was 1.5GB,
which is over 500MB larger than a new clone. So I started
experimenting with the other repack flags, including `-F`. The result
was that the repo ballooned to over 4GB and I couldn't find a way to
reduce the size. Even cloning the repo to another machine resulted in
a 1.5GB transfer. In the end, I ended up doing a fresh clone and
swapping the objects/pack directories.

I was able to reproduce the behavior with a fresh clone as well:

    git clone --bare git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git
    cd linux-stable
    git repack -a -d -F

In summary:

1. To create bitmaps without increasing disk space usage:

        git repack -a -d -b -l

1. I was not able to use `git repack -F` in a way that did not
   quadruple the size of the Linux kernel repo. It even caused clones
   of the repo to be larger as well

1. Git should have a warning if bitmaps are requested but cannot be
   created due to packSizeLimit restrictions. I plan to file a bug or
   make a patch.

[1]: https://git.kernel.org/cgit/git/git.git/tree/Documentation/RelNotes/2.0.0.txt
[2]: https://launchpad.net/~git-core/+archive/ubuntu/ppa
