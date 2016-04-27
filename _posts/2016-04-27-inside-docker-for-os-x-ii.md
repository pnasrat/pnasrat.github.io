---
layout: post
title: "Inside Docker for OS X ii"
description: "Getting a shell"
category: 
tags: ["docker"]
---
{% include JB/setup %}
## Overview

```
$ cat  /Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/xhyve.args
-A -m 2G -c 2 -u -s 0:0,hostbridge -s 31,lpc -s 2:0,virtio-ipc,uuid="e34e7389-c6ba-4f70-9f12-3a63edc453e6",path="/var/tmp/com.docker.vmnetd.socket",macfile="/Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/mac.0" -s 3:0,virtio-ipc,uuid="2954abe4-b876-45d0-948c-1b1c51e80bdb",path="/var/tmp/com.docker.slirp.socket",macfile="/Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/mac.1" -s 4,virtio-blk,'file:///Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/Docker.qcow2' -s 5,virtio-9p,path="/var/tmp/com.docker.db.socket",tag=db -s 6,virtio-rnd -s 7,virtio-9p,path=/var/tmp/com.docker.port.socket,tag=port -s 8,virtio-sock,guest_cid=3,path='/var/tmp/com.docker.vsock',guest_forwards=2376;1525 -l com1,pty="/Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty",log="/Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/console-ring" -f 'kexec,/Applications/Docker.app/Contents/Resources/moby/vmlinuz64,/Applications/Docker.app/Contents/Resources/moby/initrd.img,earlyprintk=serial console=ttyS0 com.docker.driverDir="/Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux", com.docker.database="com.docker.driver.amd64-linux"'
```

Of those pty looks interesting to me and luckyily our old friend screen knows about serial ports.

```
$ ls -l /Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty
lrwxr-xr-x  1 pnasrat  staff  12 27 Apr 13:07 /Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty -> /dev/ttys009
$ screen /dev/ttys009
```

This gives us a login prompt which handily we can just login as root

```
Welcome to Moby alpha
Kernel 4.4.6 on an x86_64 (/dev/ttyS0)

                        ##         .
                  ## ## ##        ==
               ## ## ## ## ##    ===
           /"""""""""""""""""___/ ===
      ~~~ {~~ ~~~~ ~~~ ~~~~ ~~~ ~ /  ===- ~~~
           \______ o           __/
             \    \         __/
              \____\_______/

docker login: root
Welcome to the Moby alpha, based on Alpine Linux.
docker:~#
```

```
docker:~# cat /proc/mounts
tmpfs / tmpfs rw,relatime 0 0
proc /proc proc rw,nosuid,nodev,noexec,relatime 0 0
tmpfs /run tmpfs rw,nodev,relatime,size=205104k,mode=755 0 0
binfmt_misc /proc/sys/fs/binfmt_misc binfmt_misc rw,relatime 0 0
dev /dev tmpfs rw,nosuid,relatime,size=10240k,mode=755 0 0
mqueue /dev/mqueue mqueue rw,nosuid,nodev,noexec,relatime 0 0
devpts /dev/pts devpts rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000 0 0
shm /dev/shm tmpfs rw,nosuid,nodev,noexec,relatime 0 0
sysfs /sys sysfs rw,nosuid,nodev,noexec,relatime 0 0
securityfs /sys/kernel/security securityfs rw,nosuid,nodev,noexec,relatime 0 0
debugfs /sys/kernel/debug debugfs rw,nosuid,nodev,noexec,relatime 0 0
fusectl /sys/fs/fuse/connections fusectl rw,nosuid,nodev,noexec,relatime 0 0
cgroup_root /sys/fs/cgroup tmpfs rw,nosuid,nodev,noexec,relatime,size=10240k,mode=755 0 0
openrc /sys/fs/cgroup/openrc cgroup rw,nosuid,nodev,noexec,relatime,release_agent=/lib/rc/sh/cgroup-release-agent.sh,name=openrc 0 0
cpuset /sys/fs/cgroup/cpuset cgroup rw,nosuid,nodev,noexec,relatime,cpuset 0 0
cpu /sys/fs/cgroup/cpu cgroup rw,nosuid,nodev,noexec,relatime,cpu 0 0
cpuacct /sys/fs/cgroup/cpuacct cgroup rw,nosuid,nodev,noexec,relatime,cpuacct 0 0
blkio /sys/fs/cgroup/blkio cgroup rw,nosuid,nodev,noexec,relatime,blkio 0 0
memory /sys/fs/cgroup/memory cgroup rw,nosuid,nodev,noexec,relatime,memory 0 0
devices /sys/fs/cgroup/devices cgroup rw,nosuid,nodev,noexec,relatime,devices 0 0
freezer /sys/fs/cgroup/freezer cgroup rw,nosuid,nodev,noexec,relatime,freezer 0 0
net_cls /sys/fs/cgroup/net_cls cgroup rw,nosuid,nodev,noexec,relatime,net_cls 0 0
perf_event /sys/fs/cgroup/perf_event cgroup rw,nosuid,nodev,noexec,relatime,perf_event 0 0
net_prio /sys/fs/cgroup/net_prio cgroup rw,nosuid,nodev,noexec,relatime,net_prio 0 0
hugetlb /sys/fs/cgroup/hugetlb cgroup rw,nosuid,nodev,noexec,relatime,hugetlb 0 0
pids /sys/fs/cgroup/pids cgroup rw,nosuid,nodev,noexec,relatime,pids 0 0
tracefs /sys/kernel/debug/tracing tracefs rw,nosuid,nodev,noexec,relatime 0 0
/dev/vda2 /var ext4 rw,relatime,data=ordered 0 0
db /Database 9p rw,sync,dirsync,relatime,trans=virtio,dfltuid=1001,dfltgid=50,version=9p2000 0 0
osxfs /Mac fuse.osxfs rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other,max_read=1048576 0 0
port /port 9p rw,sync,dirsync,relatime,trans=virtio,dfltuid=1001,dfltgid=50,version=9p2000 0 0
osxfs /var/log fuse.osxfs rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other,max_read=1048576 0 0
osxfs /Users fuse.osxfs rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other,max_read=1048576 0 0
osxfs /Volumes fuse.osxfs rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other,max_read=1048576 0 0
osxfs /tmp fuse.osxfs rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other,max_read=1048576 0 0
osxfs /private fuse.osxfs rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other,max_read=1048576 0 0
/dev/vda2 /var/lib/docker/aufs ext4 rw,relatime,data=ordered 0 0
```

What is interesting is that we are using a 9p mount for the Database here we can see the docker daemon running:

```
 1283 root       0:03 /usr/bin/docker daemon --pidfile=/run/docker.pid -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --config-file=/Database/branch/master/ro/com.docker.driver.amd64-linux/etc/docker/daemon.json
 ```
