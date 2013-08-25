---
layout: post
title: "Inside docker I"
description: "Setup and API exploration"
category: 
tags: docker
---
{% include JB/setup %}
## Overview

This is an atypical introduction to docker. It aims to learn more about the internals of lxc and aufs by experimenting and analyzing what docker is doing behind the scenes. The inspiration for this post comes from my favourite article from sysadvent - [Down the ls Rabbit Hole](http://sysadvent.blogspot.com/2010/12/day-15-down-ls-rabbit-hole.html). It reflects how I like to try to get understanding of various systems from a black box/trace approach before diving into the code. 

Docker has had a lot of attention, and I wanted to get a deeper understanding of how it works. I'm starting with pretty minimal knowledge on how docker works, I've done the introduction tutorial and attended a Meetup where I saw a 101. I know it uses lxc and aufs under the hood but I've not done much with those at all. I wanted to see if we can understand what's going on using standard Linux system tools.

## Requirements

Pre-requisites - install virtualbox and vagrant as per the docker installatino

* http://docs.docker.io/en/latest/installation/vagrant/

Create a clone of the main docker repo, we won't use it right away but in a later post I intend to examine the source some more.

```
git clone https://github.com/dotcloud/docker.git
cd docker
vagrant up
```

If you've not used docker before take some time to run the [hello world example](http://docs.docker.io/en/latest/examples/hello_world/#hello-world). I'm currently assuming you're running as root on the vagrant image (eg `sudo -s`).

```
docker run ubuntu /bin/echo hello world
hello world
```

## Going deeper

Lets take a closer look at the Hello World example by using our old friend `strace`.

```
strace -e trace=file,network,write -s2048  -tt -fF -o /tmp/docker.out docker run ubuntu /bin/echo hello world
```

This is what I get:

```
1782  10:45:58.022405 execve("/usr/bin/docker", ["docker", "run", "ubuntu", "/bin/echo", "hello", "world"], [/* 20 vars */]) = 0
1782  10:45:58.135032 open("/proc/sys/net/core/somaxconn", O_RDONLY|O_CLOEXEC) = 3
1782  10:45:58.135549 read(3, "128\n", 4096) = 4
1782  10:45:58.136000 read(3, "", 4092) = 0
1782  10:45:58.136823 socket(PF_INET6, SOCK_STREAM, IPPROTO_TCP) = 3
1782  10:45:58.137154 setsockopt(3, SOL_IPV6, IPV6_V6ONLY, [0], 4) = 0
1782  10:45:58.137356 bind(3, {sa_family=AF_INET6, sin6_port=htons(0), inet_pton(AF_INET6, "::1", &sin6_addr), sin6_flowinfo=0, sin6_scope_id=0}, 28) = 0
1782  10:45:58.138761 socket(PF_INET6, SOCK_STREAM, IPPROTO_TCP) = 4
1782  10:45:58.139251 setsockopt(4, SOL_IPV6, IPV6_V6ONLY, [0], 4) = 0
1782  10:45:58.139782 bind(4, {sa_family=AF_INET6, sin6_port=htons(0), inet_pton(AF_INET6, "::ffff:127.0.0.1", &sin6_addr), sin6_flowinfo=0, sin6_scope_id=0}, 28) = 0
1782  10:45:58.272733 stat("/usr/local/sbin/docker", 0xc2000d8000) = -1 ENOENT (No such file or directory)
1782  10:45:58.273302 stat("/usr/local/bin/docker", 0xc2000d8090) = -1 ENOENT (No such file or directory)
1782  10:45:58.274013 stat("/usr/sbin/docker", 0xc2000d8120) = -1 ENOENT (No such file or directory)
1782  10:45:58.274566 stat("/usr/bin/docker", {st_mode=S_IFREG|0755, st_size=5770760, ...}) = 0
1782  10:45:58.275128 stat("/usr/local/sbin/docker", 0xc2000d8240) = -1 ENOENT (No such file or directory)
1782  10:45:58.275643 stat("/usr/local/bin/docker", 0xc2000d82d0) = -1 ENOENT (No such file or directory)
1782  10:45:58.276159 stat("/usr/sbin/docker", 0xc2000d8360) = -1 ENOENT (No such file or directory)
1782  10:45:58.276577 stat("/usr/bin/docker", {st_mode=S_IFREG|0755, st_size=5770760, ...}) = 0
1782  10:45:58.278058 stat("/home/vagrant/.dockercfg", 0xc2000d8480) = -1 ENOENT (No such file or directory)
1782  10:45:58.279033 socket(PF_FILE, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, 0) = 3
1782  10:45:58.279570 setsockopt(3, SOL_SOCKET, SO_BROADCAST, [1], 4) = 0
1782  10:45:58.281848 connect(3, {sa_family=AF_FILE, path="/var/run/docker.sock"}, 23) = 0
1782  10:45:58.282319 getsockname(3, {sa_family=AF_FILE, NULL}, [2]) = 0
1782  10:45:58.282688 getpeername(3, {sa_family=AF_FILE, path="/var/run/docker.sock"}, [23]) = 0
1782  10:45:58.283017 write(3, "POST /v1.4/containers/create HTTP/1.1\r\nHost: \r\nUser-Agent: Docker-Client/0.5.3\r\nContent-Length: 335\r\nContent-Type: application/json\r\n\r\n{\"Hostname\":\"\",\"User\":\"\",\"Memory\":0,\"MemorySwap\":0,\"CpuShares\":0,\"AttachStdin\":false,\"AttachStdout\":true,\"AttachStderr\":true,\"PortSpecs\":null,\"Tty\":false,\"OpenStdin\":false,\"StdinOnce\":false,\"Env\":null,\"Cmd\":[\"/bin/echo\",\"hello\",\"world\"],\"Dns\":null,\"Image\":\"ubuntu\",\"Volumes\":{},\"VolumesFrom\":\"\",\"Entrypoint\":[],\"NetworkDisabled\":false}", 470) = 470
1782  10:45:58.284722 read(3, "HTTP/1.1 201 Created\r\nContent-Type: text/plain; charset=utf-8\r\nContent-Length: 21\r\nDate: Sun, 25 Aug 2013 10:45:58 GMT\r\n\r\n{\"Id\":\"6b4b8793687d\"}", 4096) = 143
1782  10:45:58.286191 socket(PF_FILE, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, 0) = 3
1782  10:45:58.286529 setsockopt(3, SOL_SOCKET, SO_BROADCAST, [1], 4) = 0
1782  10:45:58.287292 connect(3, {sa_family=AF_FILE, path="/var/run/docker.sock"}, 23) = 0
1782  10:45:58.287629 getsockname(3, {sa_family=AF_FILE, NULL}, [2]) = 0
1782  10:45:58.287989 getpeername(3, {sa_family=AF_FILE, path="/var/run/docker.sock"}, [23]) = 0
1782  10:45:58.288605 write(3, "POST /v1.4/containers/6b4b8793687d/start HTTP/1.1\r\nHost: \r\nUser-Agent: Docker-Client/0.5.3\r\nContent-Length: 35\r\nContent-Type: application/json\r\n\r\n{\"Binds\":null,\"ContainerIDFile\":\"\"}", 181) = 181
1782  10:45:58.290946 read(3, 0xc200132000, 4096) = -1 EAGAIN (Resource temporarily unavailable)
1782  10:45:58.293395 read(3, "HTTP/1.1 204 No Content\r\nContent-Type: text/plain; charset=utf-8\r\nContent-Length: 0\r\nDate: Sun, 25 Aug 2013 10:45:58 GMT\r\n\r\n", 4096) = 124
1782  10:45:58.294246 socket(PF_FILE, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, 0) = 3
1782  10:45:58.294400 setsockopt(3, SOL_SOCKET, SO_BROADCAST, [1], 4) = 0
1782  10:45:58.295089 connect(3, {sa_family=AF_FILE, path="/var/run/docker.sock"}, 23) = 0
1782  10:45:58.295432 getsockname(3, {sa_family=AF_FILE, NULL}, [2]) = 0
1782  10:45:58.295774 getpeername(3, {sa_family=AF_FILE, path="/var/run/docker.sock"}, [23]) = 0
1782  10:45:58.296514 write(3, "POST /v1.4/containers/6b4b8793687d/attach?logs=1&stderr=1&stdout=1&stream=1 HTTP/1.1\r\nHost: \r\nUser-Agent: Docker-Client/0.5.3\r\nContent-Length: 0\r\nContent-Type: plain/text\r\n\r\n", 174) = 174
1782  10:45:58.297613 read(3, 0xc200135000, 4096) = -1 EAGAIN (Resource temporarily unavailable)
1782  10:45:58.298491 read(3, "HTTP/1.1 200 OK\r\nContent-Type: application/vnd.docker.raw-stream\r\n\r\n", 4096) = 68
1782  10:45:58.298962 write(1, "", 0)   = 0
1782  10:45:58.299223 read(3, 0xc200135000, 4096) = -1 EAGAIN (Resource temporarily unavailable)
1782  10:45:58.301381 read(0,  <unfinished ...>
1786  10:45:58.463570 read(3, "hello world\n", 4096) = 12
1786  10:45:58.464443 write(1, "hello world\n", 12) = 12
1786  10:45:58.465872 read(3, 0xc200135000, 4096) = -1 EAGAIN (Resource temporarily unavailable)
1788  10:45:58.475465 read(3, "", 4096) = 0
```

As we can see we have a REST API over a unix socket. The REST API is lnked to from the [docs home page](http://docs.docker.io/en/latest/).

The API is fully documented, from the strace output we can see that we're using [1.4](http://docs.docker.io/en/latest/api/docker_remote_api_v1.4/)

Sadly this means we can't use curl in the default configuration. You can enable an HTTP connection via a command line flag.

In our git repository lets see if we can figure out what is going on

```
 git grep /usr/bin/docker
 hack/release/make.sh:    /usr/bin/docker -d
 hack/release/make.sh:   cp bundles/$VERSION/binary/docker-$VERSION $DIR/usr/bin/docker
 packaging/ubuntu/docker.upstart:    /usr/bin/docker -d
 packaging/ubuntu/lxc-docker.prerm:if [ "`pgrep -f '/usr/bin/docker -d'`" != "" ]; then /sbin/stop docker; fi
```

Reading the sources we can run a port so lets edit `/etc/init/docker.conf` and change the script to

```
    /usr/bin/docker -d -H 127.0.0.1:4243
```