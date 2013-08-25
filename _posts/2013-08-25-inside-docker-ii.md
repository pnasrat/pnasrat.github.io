---
layout: post
title: "inside docker ii"
description: ""
category: 
tags: docker
---
{% include JB/setup %}

## Further down the rabbit hole

In our previous post we explored the hello world example, learning what happens when it is executed and how docker talks to itself. This is interesting but doesn't give us much information about what is going on. Lets launch a daemon instead. After a bit of trial and error I've opted to use http.server (aka SimpleHTTPServer) from python stdlib.

```
docker run -d -p 8000:8000 -t ubuntu:12.10  /usr/bin/python3 -m http.server
docker ps
ID                  IMAGE               COMMAND                CREATED              STATUS              PORTS
a86b320f3c71        ubuntu:12.10        /usr/bin/python3 -m    About a minute ago   Up About a minute   8000->8000
lsof -nP -i :8000
COMMAND PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
docker  473 root    5u  IPv4  14398      0t0  TCP 127.0.0.1:8000 (LISTEN)
```

Process 473 is the docker daemon process, lets try understand it a bit more.

```
ls -l /proc/473/fd
total 0
lrwx------ 1 root root 64 Aug 25 10:53 0 -> /dev/null
lrwx------ 1 root root 64 Aug 25 10:53 1 -> /dev/pts/7
lrwx------ 1 root root 64 Aug 25 10:53 10 -> /dev/ptmx
lrwx------ 1 root root 64 Aug 25 10:53 2 -> /dev/pts/7
lrwx------ 1 root root 64 Aug 25 10:53 3 -> socket:[7643]
lrwx------ 1 root root 64 Aug 25 10:53 4 -> anon_inode:[eventpoll]
lrwx------ 1 root root 64 Aug 25 10:53 5 -> socket:[14398]
lr-x------ 1 root root 64 Aug 25 10:53 7 -> /dev/urandom
lrwx------ 1 root root 64 Aug 25 10:53 8 -> /var/lib/docker/containers/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793-json.log
lrwx------ 1 root root 64 Aug 25 10:53 9 -> /var/lib/docker/containers/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793-json.log
```

Lets see what that log is

```
cat /var/lib/docker/containers/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793-json.log
{"log":"Serving HTTP on 0.0.0.0 port 8000 ...\r\n","stream":"stdout","time":"2013-08-25T10:52:22.257575646Z"}{"log":"172.17.42.1 - - [25/Aug/2013 10:53:10] \"GET / HTTP/1.1\" 200 -\r\n","stream":"stdout","time":"2013-08-25T10:53:10.572209376Z"}
```

We know we're using containers so the process should be visible to us - use your favourite mechanism to track it down - eg

```
pgrep python
2308
ps -fp 2308
UID        PID  PPID  C STIME TTY          TIME CMD
root      2308  2306  0 10:52 ?        00:00:00 /usr/bin/python3 -m http.server
```

So now lets compare what we have going on in the daemon process

```
ls -l /proc/2308/fd
total 0
lr-x------ 1 root root 64 Aug 25 10:53 0 -> /dev/null
lrwx------ 1 root root 64 Aug 25 10:53 1 -> /dev/pts/1
lrwx------ 1 root root 64 Aug 25 10:53 2 -> /dev/pts/1
lrwx------ 1 root root 64 Aug 25 10:53 3 -> socket:[14540]
lrwx------ 1 root root 64 Aug 25 10:53 8 -> /var/lib/docker/containers/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793/rootfs.hold
file /var/lib/docker/containers/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793/rootfs.hold
/var/lib/docker/containers/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793/rootfs.hold: empty
```

So we've found where docker is keeping it's containers, and where  root filesystem is for this system.

```
ls -l /var/lib/docker/containers/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793/
total 24
-rw-------  1 root root  967 Aug 25 10:56 a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793-json.log
-rw-r--r--  1 root root  985 Aug 25 10:52 config.json
-rw-r--r--  1 root root 3571 Aug 25 10:52 config.lxc
-rw-r--r--  1 root root   35 Aug 25 10:52 hostconfig.json
drwxr-xr-x 28 root root 4096 Aug 25 10:52 rootfs
-rw-------  1 root root    0 Aug 25 10:52 rootfs.hold
drwxr-xr-x  4 root root 4096 Aug 25 10:52 rw
```

The json files are the files docker uses itself, then it generates the lxc specific configuration and we also have a .hold file which seems to relate to [rootfs pinning](http://www.mail-archive.com/lxc-users@lists.sourceforge.net/msg03536.html).


```
ls -l /var/lib/docker/containers/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793/rootfs
total 80
drwxr-xr-x  2 root root 4096 Jan 25  2013 bin
drwxr-xr-x  2 root root 4096 Oct  9  2012 boot
drwxr-xr-x  3 root root 4096 Aug 25 10:38 dev
drwxr-xr-x 60 root root 4096 Jan 25  2013 etc
drwxr-xr-x  2 root root 4096 Oct  9  2012 home
drwxr-xr-x 12 root root 4096 Aug 25 10:38 lib
drwxr-xr-x  2 root root 4096 Aug 25 10:39 lib64
drwxr-xr-x  2 root root 4096 Jan 25  2013 media
drwxr-xr-x  2 root root 4096 Oct  9  2012 mnt
drwxr-xr-x  2 root root 4096 Jan 25  2013 opt
drwxr-xr-x  2 root root 4096 Oct  9  2012 proc
drwx------  2 root root 4096 Aug 25 10:38 root
drwxr-xr-x  6 root root 4096 Aug 25 10:39 run
drwxr-xr-x  2 root root 4096 Aug 25 10:39 sbin
drwxr-xr-x  2 root root 4096 Jun 11  2012 selinux
drwxr-xr-x  2 root root 4096 Jan 25  2013 srv
drwxr-xr-x  2 root root 4096 Jul 21  2012 sys
drwxrwxrwt  2 root root 4096 Jan 25  2013 tmp
drwxr-xr-x 10 root root 4096 Aug 25 10:38 usr
drwxr-xr-x 11 root root 4096 Aug 25 10:39 var
```

So the rootfs directory contains the full image filesystem, this is expected as we're using the base image. We'll play explore this more shortly. 

We know that we expect the usage of cgroups, so lets see what the process can reveal about that

```
cat /proc/2308/cgroup
9:hugetlb:/lxc/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793
8:perf_event:/lxc/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793
7:blkio:/lxc/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793
6:freezer:/lxc/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793
5:devices:/lxc/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793
4:memory:/lxc/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793
3:cpuacct:/lxc/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793
2:cpu:/lxc/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793
1:cpuset:/lxc/a86b320f3c71a7473d06f2184a955d7f76ef3dfb113b0f22b4a8cd8a544f0793
```

I intend to cover cgroups more fully in a follow up - for now you can whet your appetite on the [kernel docs](https://www.kernel.org/doc/Documentation/cgroups/cgroups.txt).

We know the FS is aufs, we should see that from the processes mount table.

```
cat /proc/2308/mounts
rootfs / rootfs rw 0 0
none / aufs rw,relatime,si=3a1cffc99314fd51 0 0
proc /proc proc rw,nosuid,nodev,noexec,relatime 0 0
sysfs /sys sysfs rw,nosuid,nodev,noexec,relatime 0 0
/dev/mapper/precise64-root /sbin/init ext4 ro,relatime,errors=remount-ro,data=ordered 0 0
tmpfs /run/resolvconf/resolv.conf tmpfs ro,relatime,size=74716k,mode=755 0 0
devpts /dev/tty1 devpts rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000 0 0
devpts /dev/pts devpts rw,relatime,mode=600,ptmxmode=666 0 0
devpts /dev/ptmx devpts rw,relatime,mode=600,ptmxmode=666 0 0
```

So from a small bit of poking we understand more about the layout of lxc and how things appear to the processes in the container.
