---
layout: post
title: "inside docker iii"
description: ""
category: 
tags: docker
---
{% include JB/setup %}

I took a brief hiatus from my internal spelunking to look at how image creation might work. On IRC someone mentioned it'd be nice to have a Centos 5 image. Knowing a little about how anaconda and RPM work internally I thought it'd make a fun distraction. Let's see what I discovered along the way.

So a basic clean docker setup, grab the Centos 6 Image.

```
$ docker run -i  -t centos /bin/bash
```

I started looking at what they were using. I'm of the opinion that docker images should be minimal. The first thought was to see what the base image was made of in the Centos 6 image.

```
rpm -Va
S.5....T.    /sbin/init
.M.......    /
.M.......    /sys
```

If we remind ourself about `rpm` verification from [Maximum RPM](http://www.rpm.org/max-rpm/s1-rpm-verify-output.html). M is the file's mode, the difference between a container and it's outside host would impact that for `/` and `/sys`.

Let's take a closer look at `init` though, the environment is pretty minimal (no `FILE(1)`) so let us exit then switch 

```
docker ps -a
ID                  IMAGE               COMMAND                CREATED             STATUS              PORTS
e30e657a24c1        centos:6.4          /bin/bash              8 minutes ago       Exit 127
```

We know from last time where we expect the rootfs to be. 

```
# cd /var/lib/docker/containers/e30e657a24c1*
# echo $PWD
/var/lib/docker/containers/e30e657a24c192694d06a41c4e201576a52d59596aa364ac7aa375d8c2696f09
```

Lets take a look at the dir again

```
-rw-r--r-- 1 root root   854 Aug 27 01:17 config.json
-rw-r--r-- 1 root root  3570 Aug 27 01:08 config.lxc
-rw------- 1 root root 13545 Aug 27 01:17 e30e657a24c192694d06a41c4e201576a52d59596aa364ac7aa375d8c2696f09-json.log
-rw-r--r-- 1 root root    35 Aug 27 01:08 hostconfig.json
-rw------- 1 root root     0 Aug 27 01:08 rootfs.hold
drwxr-xr-x 5 root root  4096 Aug 27 01:17 rw
```

So this time we don't see the `rootfs` dir but we do see an `rw` dir - lets look in that.

```
# find rw
rw
rw/.wh..wh.plnk
rw/var
rw/var/lib
rw/var/lib/rpm
rw/var/lib/rpm/__db.002
rw/var/lib/rpm/__db.003
rw/var/lib/rpm/__db.004
rw/var/lib/rpm/__db.001
rw/.bash_history
rw/.wh..wh.aufs
rw/.wh..wh.orph
```
So we're starting to find out about aufs here - this is an incremental change - the Berkeley DB region (the `__db.*` files), the `.bash_history` files we expect to have changes and be the delta. The `.wh..wh.*` files are interesting though. This is our first insight into how aufs is working under the hood.

So we know it's an overlay based filesystem so we expect a delta clearly there is some metadata here too:

```
find . -name .wh\* | xargs file
./rw/.wh..wh.plnk: directory
./rw/.wh..wh.aufs: empty
./rw/.wh..wh.orph: directory
```

To find out more we can try figure out more information.

```
modinfo aufs
filename:       /lib/modules/3.8.0-19-generic/kernel/ubuntu/aufs/aufs.ko
version:        3.x-rcN-20121112
description:    aufs -- Advanced multi layered unification filesystem
author:         Junjiro R. Okajima <aufs-users@lists.sourceforge.net>
license:        GPL
srcversion:     24B5792563ECCFC0A3F445B
depends:
intree:         Y
vermagic:       3.8.0-19-generic SMP mod_unload modversions
parm:           brs:use <sysfs>/fs/aufs/si_*/brN (int)
```

So lets figure out where the kmod comes from:

```
dpkg -S /lib/modules/3.8.0-19-generic/kernel/ubuntu/aufs/aufs.ko
linux-image-3.8.0-19-generic: /lib/modules/3.8.0-19-generic/kernel/ubuntu/aufs/aufs.ko
```

OK so we have a package name - I'm not a dpkg expert but I think we can do:

```
dpkg -s linux-image-3.8.0-19-generic | grep Source
Source: linux-lts-raring
```

I'm not quite up to date so have to go get the appropriate bits via http://packages.ubuntu.com/raring/linux-image-3.8.0-19-generic and eventually do `dpkg-source -x linux_3.8.0-29.42.dsc`. A bit of poking around and we can find in `./ubuntu/include/uapi/linux/aufs_type.h`

```c
#define AUFS_PLINKDIR_NAME      AUFS_WH_PFX "plnk"
#define AUFS_ORPHDIR_NAME       AUFS_WH_PFX "orph"
```

Then also `./ubuntu/aufs/whout.c` there is a lot of interest here just reading the comments 

```
grep todo ./ubuntu/aufs/whout.c
/* todo: should this mkdir be done in /sbin/mount.aufs helper? */
	 * todo: should this create be done in /sbin/mount.aufs helper?
```

I think there is quite a lot of further digging to be done in the internals here.

