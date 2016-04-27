---
layout: post
title: "Insider Docker for OS X i"
description: "Initial exploration"
category: 
tags: "docker"
---
{% include JB/setup %}
## Overview

This is another atypical introduction to docker on OS X a follow up to my previous posts. It's been 3 years since I blogged, I've been busy, burnt out and a lot has changed in my life. However I thought this would be a good thing to explore.

It aims to learn more about the internals of xhyve and the new docker native beta on OS X by experimenting and analyzing what docker is doing behind the scenes. Like my previous posts the inspiration for this post comes from my favourite article from sysadvent - [Down the ls Rabbit Hole](http://sysadvent.blogspot.com/2010/12/day-15-down-ls-rabbit-hole.html).

## Requirements

Pre-requisites - access to the Docker beta.

## What do we have

So lets see what we start off with

```
$ docker info
Client:
 Version:      1.11.0
 API version:  1.23
 Go version:   go1.5.4
 Git commit:   4dc5990
 Built:        Wed Apr 13 19:36:04 2016
 OS/Arch:      darwin/amd64

Server:
 Version:      1.11.0
 API version:  1.23
 Go version:   go1.5.4
 Git commit:   a5315b8
 Built:        Mon Apr 18 19:19:21 2016
 OS/Arch:      linux/amd64

$ docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 2
Server Version: 1.11.0
Storage Driver: aufs
 Root Dir: /var/lib/docker/aufs
 Backing Filesystem: extfs
 Dirs: 1
 Dirperm1 Supported: true
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins: 
 Volume: local
 Network: host bridge null
Kernel Version: 4.4.6
Operating System: Alpine Linux v3.3
OSType: linux
Architecture: x86_64
CPUs: 2
Total Memory: 1.956 GiB
Name: docker
ID: SGTB:RMFN:XSDC:P5SK:CTPV:IBBO:QFOE:M6FL:OG77:CXZL:ZBVS:HRDM
Docker Root Dir: /var/lib/docker
Debug mode (client): false
Debug mode (server): true
 File Descriptors: 15
 Goroutines: 33
 System Time: 2016-04-27T17:28:52.124543023Z
 EventsListeners: 1
Registry: https://index.docker.io/v1/
```

OK so it looks as if we are talking to a server running Alpine Linux on OS X. 

Lets just see what is running on our OS X host using our old friend `ps`.

```
pnasrat         32157   0.2  0.1 573469540  11092   ??  Ss   10:05am   0:13.15 /Applications/Docker.app/Contents/MacOS/com.docker.driver.amd64-linux -db /var/tmp/com.docker.db.socket -osxfs-volume /var/tmp/com.docker.osxfs.volume.socket -slirp /var/tmp/com.docker.slirp.socket -vmnet /var/tmp/com.docker.vmnetd.socket -port /var/tmp/com.docker.port.socket -vsock /var/tmp/com.docker.vsock -docker /var/tmp/docker.sock -addr fd:3 -debug
pnasrat         32144   0.1  0.0 573414380   4068   ??  S    10:04am   0:08.44 /Applications/Docker.app/Contents/MacOS/com.docker.osx.xhyve.linux -watchdog fd:0
pnasrat         32158   0.1  0.1 573412816   5012   ??  S    10:05am   0:08.44 /Applications/Docker.app/Contents/MacOS/com.docker.driver.amd64-linux -db /var/tmp/com.docker.db.socket -osxfs-volume /var/tmp/com.docker.osxfs.volume.socket -slirp /var/tmp/com.docker.slirp.socket -vmnet /var/tmp/com.docker.vmnetd.socket -port /var/tmp/com.docker.port.socket -vsock /var/tmp/com.docker.vsock -docker /var/tmp/docker.sock -addr fd:3 -debug
root            32161   0.0  0.0  2469532    648   ??  S    10:05am   0:00.07 /Library/PrivilegedHelperTools/com.docker.vmnetd
pnasrat         32160   0.0  0.1 573422672   4992   ??  S    10:05am   0:00.02 /Applications/Docker.app/Contents/MacOS/com.docker.driver.amd64-linux -xhyve /Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/xhyve.args
pnasrat         32159   0.0  0.1 575529704   5364   ??  S    10:05am   0:00.05 /Applications/Docker.app/Contents/MacOS/com.docker.driver.amd64-linux -xhyve /Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/xhyve.args
pnasrat         32156   0.0  0.0  2469204   2132   ??  Ss   10:05am   0:00.03 com.docker.slirp --db /var/tmp/com.docker.db.socket --socket fd:3 --port-control fd:4
pnasrat         32155   0.0  0.0  2467860   3604   ??  Ss   10:05am   0:00.02 com.docker.osxfs --address fd:3 --connect /var/tmp/com.docker.vsock/connect --volume-control fd:4 --path /
pnasrat         32150   0.0  0.0 573405164   4008   ??  S    10:04am   0:00.02 /Applications/Docker.app/Contents/MacOS/com.docker.osx.xhyve.linux
pnasrat         32148   0.0  0.1 573426064   5532   ??  Ss   10:04am   0:01.62 com.docker.backend
pnasrat         32145   0.0  0.1  2562624  10816   ??  Ss   10:04am   0:02.37 com.docker.db --url=file:///var/tmp/com.docker.db.socket --git /Users/pnasrat/Library/Containers/com.docker.docker/Data/database
pnasrat         32143   0.0  0.1 573453428   5628   ??  S    10:04am   0:00.10 /Applications/Docker.app/Contents/MacOS/com.docker.osx.xhyve.linux -watchdog fd:0
root            32140   0.0  0.0  2436148   2140   ??  Ss   10:04am   0:00.02 /Library/PrivilegedHelperTools/com.docker.vmnetd
```

We can inspect the logs using `syslog -k Sender Docker`

```
Apr 27 13:07:17 enki Docker[44922] <Notice>: filesystem: osxfs
Apr 27 13:07:17 enki Docker[44922] <Notice>: Hypervisor: native; BootProtocol: direct; UefiBootDisk: /Users/pnasrat/UefiBoot.qcow2
Apr 27 13:07:17 enki Docker[44922] <Notice>: Docker is not responding: waiting 0.5s
Apr 27 13:07:17 enki Docker[44925] <Notice>: exec: /Applications/Docker.app/Contents/MacOS/com.docker.driver.amd64-linux []string{"-A", "-m", "2G", "-c", "2", "-u", "-s", "0:0,hostbridge", "-s", "31,lpc", "-s", "2:0,virtio-ipc,uuid=e34e7389-c6ba-4f70-9f12-3a63edc453e6,path=/var/tmp/com.docker.vmnetd.socket,macfile=/Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/mac.0", "-s", "3:0,virtio-ipc,uuid=2954abe4-b876-45d0-948c-1b1c51e80bdb,path=/var/tmp/com.docker.slirp.socket,macfile=/Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/mac.1", "-s", "4,virtio-blk,file:///Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/Docker.qcow2", "-s", "5,virtio-9p,path=/var/tmp/com.docker.db.socket,tag=db", "-s", "6,virtio-rnd", "-s", "7,virtio-9p,path=/var/tmp/com.docker.port.socket,tag=port", "-s", "8,virtio-sock,guest_cid=3,path=/var/tmp/com.docker.vsock,guest_forwards=2376;1525", "-l", "com1,pty=/Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty,log=/Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/console-ring", "-f", "kexec,/Applications/Docker.app/Contents/Resources/moby/vmlinuz64,/Applications/Docker.app/Contents/Resources/moby/initrd.img,earlyprintk=serial console=ttyS0 com.docker.driverDir=\"/Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux\", com.docker.database=\"com.docker.driver.amd64-linux\""}
Apr 27 13:07:17 enki Docker[44927] <Notice>: Client reports version 12, commit 3c1bfeb0e86a9403f82302edfea4c4987cc2cb32
Apr 27 13:07:17 enki Docker[44927] <Notice>: com.docker.vmnetd reports MAC address: 22:b9:2e:1e:cf:32
Apr 27 13:07:17 enki Docker[44927] <Notice>: rx batching enabled (len=64)
Apr 27 13:07:20 enki Docker[44922] <Notice>: Docker is not responding: waiting 0.5s
--- last message repeated 8 times ---
Apr 27 13:07:26 enki Docker[44922] <Notice>: Docker is responding
```

## Components of Docker for OS X

* `pinata` -  manage the Docker.app configuration database

Lets see what this returns - the default output to terminal is fancy but this does the right thing `pinata list | pbcopy`

```
These are advanced configuration settings to customise Docker.app on MacOSX.
You can set them via pinata set <key> <value> <options>.

*  hostname = docker
   Hostname of the virtual machine endpoint, where container ports will be
   exposed if using nat networking. Access it via 'docker.local'.

*  hypervisor = native (memory=2, ncpu=2)
   The Docker.app includes embedded hypervisors that run the virtual machines
   that power the containers. This setting allows you to control which the
   default one used for Linux is.

 >  native: a version of the xhyve hypervisor that uses the MacOSX
            Hypervisor.framework to run container VMs. Parameters: memory (VM
            memory in gigabytes), ncpu (vCPUs)


*  network = nat (external-bind=false)
   Controls how local containers can access the external network via the
   MacOS X host. This includes outbound traffic as well as publishing ports
   for external access to the local containers.

 >     nat: a mode that uses the MacOS X vmnet.framework to route container
            traffic to the host network via a NAT. Parameters:
            external-bind (bind ports to external network interface)
 > hostnet: a mode that helps if you are using a VPN that restricts
            connectivity. Activating this mode will proxy container network
            packets via the Docker.app process as host socket traffic.
            Parameters: docker-ipv4 (docker node), host-ipv4 (host node)

*  filesystem = osxfs 
   Controls the mode by which files from the MacOS X host and the container
   filesystem are shared with each other.

 >   osxfs: a FUSE-based filesystem that bidirectionally forwards OSX
            filesystem events into the container. 


*  daemon = run 'pinata get daemon' or 'pinata set daemon [@file|-]>
   JSON configuration of the local Docker daemon. Configure any custom
   options you need as documented in:
   https://docs.docker.com/engine/reference/commandline/daemon/. Set it
   directly, or a @file or - for stdin.
```

From the process listing we can see this uses `/var/tmp/com.docker.db.socket`

Running `pinata` go get the help shows us we need to look in `--db-path` which is defaulting to 
`/Users/pnasrat/Library/Containers/com.docker.docker/Data/database/`

```
$ ls -al
total 0
drwxr-xr-x   4 pnasrat  staff  136 27 Apr 09:02 .
drwxr-xr-x   8 pnasrat  staff  272 27 Apr 13:07 ..
drwxr-xr-x   8 pnasrat  staff  272 27 Apr 13:21 .git
drwxr-xr-x  17 pnasrat  staff  578 27 Apr 10:04 com.docker.driver.amd64-linux
$ git show
commit 83b0394b2e88f919861e7676ad7cb8a38019f698
Author: irmin9p <irmin@openmirage.org>
Date:   Wed Apr 27 17:07:17 2016 +0000

    (no commit message)
```

This shows us we are using [openmirage](https://mirage.io/) for some of the internals

Let us look what is here

```
$ tree
.
└── com.docker.driver.amd64-linux
    ├── etc
    │   ├── docker
    │   │   └── daemon.json
    │   ├── hostname
    │   └── sysctl.conf
    ├── expose-docker-socket
    ├── filesystem
    ├── hypervisor
    ├── insecure-registry
    ├── memory
    ├── native
    │   ├── boot-protocol
    │   ├── port-forwarding
    │   └── uefi-boot-disk
    ├── ncpu
    ├── network
    ├── on-sleep
    ├── proxy-verbose
    ├── schema-version
    ├── slirp
    │   ├── docker
    │   └── host
    ├── version
    └── vmnet-simulate-failure

5 directories, 20 files
```


## xhyve

So we see from our process listing [xhyve](https://github.com/mist64/xhyve) - this is what is providing the containerisation/virutalization analgous to lxc. This builds on a new feature in OS X from 10.10 [Hypervisor framework](https://developer.apple.com/library/mac/documentation/DriversKernelHardware/Reference/Hypervisor/index.html).

For now we can 

```
cat /Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/xhyve.args
-A -m 2G -c 2 -u -s 0:0,hostbridge -s 31,lpc -s 2:0,virtio-ipc,uuid="e34e7389-c6ba-4f70-9f12-3a63edc453e6",path="/var/tmp/com.docker.vmnetd.socket",macfile="/Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/mac.0" -s 3:0,virtio-ipc,uuid="2954abe4-b876-45d0-948c-1b1c51e80bdb",path="/var/tmp/com.docker.slirp.socket",macfile="/Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/mac.1" -s 4,virtio-blk,'file:///Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/Docker.qcow2' -s 5,virtio-9p,path="/var/tmp/com.docker.db.socket",tag=db -s 6,virtio-rnd -s 7,virtio-9p,path=/var/tmp/com.docker.port.socket,tag=port -s 8,virtio-sock,guest_cid=3,path='/var/tmp/com.docker.vsock',guest_forwards=2376;1525 -l com1,pty="/Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty",log="/Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/console-ring" -f 'kexec,/Applications/Docker.app/Contents/Resources/moby/vmlinuz64,/Applications/Docker.app/Contents/Resources/moby/initrd.img,earlyprintk=serial console=ttyS0 com.docker.driverDir="/Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux", com.docker.database="com.docker.driver.amd64-linux"'
```

I'll investigate this further in a future post but lots of useful information - eg

```
file /Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/Docker.qcow2
/Users/pnasrat/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/Docker.qcow2: Qemu Image, Format: Qcow
```

## Networking

I'll investigate this in a future post

## Filesystem

I created a simple Dockerfile to explore image and container storage

```
$ echo hi > hello
$ cat Dockerfile
FROM scratch
ADD hello /
CMD ["/hello"]
$ docker build .
```

I'll investigate this in a future post.

## Sources

As part of the beta there is a link to the GPL software used. A list of OSS licenses is in `/Applications/Docker.app/Contents/Resources/OSS-LICENSES` [sources available](https://beta.docker.com/docs/opensource/).