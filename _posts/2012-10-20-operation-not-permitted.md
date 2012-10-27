---
layout: post
title: "When root cannot delete a file"
category: 
tags: []
---

Operation not permitted

It started when dpkg could not upgrade the util-linux package because the
file /usr/bin/delpart could not be symlinked. So I tried to delete the file.

    sudo rm /usr/bin/delpart
    rm: cannot remove `/usr/bin/delpart': Operation not permitted

All operations on the file failed. I tried mv, fsck, reboot into
rescue, etc.

So I googled "linux ext4 'Operation not permitted". This did not help
much, but I noticed a link about ext2 extended attributes. I have
never used extended attributes, so I did a quick read of the man page
for lsattr and chattr.

    cd /usr/bin
    sudo lsattr delpart
    ---D-a-----tT-- delpart

That was a strange set of attributes. So I compared to another random file.

    sudo lsattr zip
    -------------e- zip

Once the problem is found, the solution is straightforward

    sudo chattr +e -DatT delpart
    sudo lsattr delpart
    -------------e- delpart

The question now is how the file got to this state. I can only
speculate that a fsck run "repaired" this corrupted file to this
strange but consistent state. I wonder if there are other surprises
waiting for me on this disk.
