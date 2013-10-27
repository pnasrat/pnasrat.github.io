---
layout: post
title: "Docker testing"
description: ""
category: 
tags: docker,golang
---
{% include JB/setup %}
The [documentation](http://docs.docker.io/en/latest/contributing/devenvironment/
) for running the Docker tests is a little sparse. I thought I'd share some snippets of things I've learnt.

## DNS server ##

tl;dr add `-dns 8.8.8.8`

This has been discussed extensively on the lists and IRC, but people still seem to have issues. If you're seeing something like this message from

```
sudo docker run -lxc-conf=lxc.aa_profile=unconfined -privileged -v `pwd`:/go/src/github.com/dotcloud/docker docker hack/make.sh test
 ```

```
Unable to pull the test image:%!(EXTRA *url.Error=Get https://index.docker.io/v1/images/docker-test-image/ancestry: lookup index.docker.io. 
```

The likely solution is to specify a publicly routable DNS server, commonly used is [Google Public DNS](https://developers.google.com/speed/public-dns/), this is generally needed if your local setup is specifying a local DNS server the container can't get to - common in vagrant setups with a local private DNS server. You can check this by running `/bin/cat /etc/resolv.conf` in the container. 

```
sudo docker run -lxc-conf=lxc.aa_profile=unconfined -privileged -v `pwd`:/go/src/github.com/dotcloud/docker docker hack/make.sh test
```

## Running a single test ##

By design, for reproducibility the build/test environment is hermetic  - the Dockerfile `add`s the source directory so it is not a volume so edits/changes require a rebuild. I found myself doing a build then shell in the container repeatedly so I simply created an alias

```
alias b='sudo docker build -t docker . && sudo docker run -dns 8.8.8.8 -lxc-conf=lxc.aa_profile=unconfined -privileged -v /vagrant:/go/src/github.com/dotcloud/docker -i -t docker /bin/bash'
```

Once you have a shell in the docker build environment you can use `go` to build and run the tests. The reason I use this when developing rather than the `make.sh` wrapper script is that it gives me a finer level of control and enables me to debug. 

It took me a bit to adjust to the [testing style](http://golang.org/doc/code.html#Testing) of go, I'm very used to xUnit style testing. The Docker tests for `container.go` basically involve setup and then performing operations on it. This is what I came up with:

```go
func TestRestartGhost(t *testing.T) {
        runtime := mkRuntime(t)
        defer nuke(runtime)

        container, err := runtime.Create(&Config{
                Image:   GetTestImage(runtime).ID,
                Cmd:     []string{"sh", "-c", "echo -n bar > /test/foo"},
                Volumes: map[string]struct{}{"/test": {}},
        },
        )

        if err != nil {
                t.Fatal(err)
        }
        if err := container.Kill(); err != nil {
                t.Fatal(err)
        }

        container.State.Ghost = true
        _, err = container.Output()

        if err != nil {
                t.Fatal(err)
        }
}
```

Now to run our test, in my case I was working on a [pull request](https://github.com/dotcloud/docker/pull/2409) to fix a bug. The three ways of testing - the full test suite command described at the start of this post. Or

```
# go test -test.run=TestRestartGhost
warning: building out-of-date packages:
        code.google.com/p/go.net/websocket
        github.com/dotcloud/tar
        github.com/dotcloud/docker/utils
        github.com/dotcloud/docker/auth
        github.com/dotcloud/docker/registry
        github.com/dotcloud/docker/term
        github.com/gorilla/context
        github.com/gorilla/mux
        github.com/kr/pty
installing these packages with 'go test -i' will speed future tests.

2013/10/27 14:55:10 Listening for HTTP on 127.0.0.1:4270 (tcp)
PASS
ok      github.com/dotcloud/docker      1.431s
```



The second option is o just build the tests:

```
# go test -c
warning: building out-of-date packages:
        code.google.com/p/go.net/websocket
        github.com/dotcloud/tar
        github.com/dotcloud/docker/utils
        github.com/dotcloud/docker/auth
        github.com/dotcloud/docker/registry
        github.com/dotcloud/docker/term
        github.com/gorilla/context
        github.com/gorilla/mux
        github.com/kr/pty
installing these packages with 'go test -i' will speed future tests.
# ls -al docker.test
-rwxr-xr-x 1 1000 1000 9961848 Oct 27 14:50 docker.test
# ./docker.test --help
2013/10/27 14:53:20 Listening for HTTP on 127.0.0.1:4270 (tcp)
Usage of ./docker.test:
  -httptest.serve="": if non-empty, httptest.NewServer serves on this address and blocks
  -test.bench="": regular expression to select benchmarks to run
  -test.benchmem=false: print memory allocations for benchmarks
  -test.benchtime=1s: approximate run time for each benchmark
  -test.blockprofile="": write a goroutine blocking profile to the named file after execution
  -test.blockprofilerate=1: if >= 0, calls runtime.SetBlockProfileRate()
  -test.cpu="": comma-separated list of number of CPUs to use for each test
  -test.cpuprofile="": write a cpu profile to the named file during execution
  -test.memprofile="": write a memory profile to the named file after execution
  -test.memprofilerate=0: if >=0, sets runtime.MemProfileRate
  -test.parallel=1: maximum test parallelism
  -test.run="": regular expression to select tests and examples to run
  -test.short=false: run smaller test suite to save time
  -test.timeout=0: if positive, sets an aggregate time limit for all tests
  -test.v=false: verbose: print additional output
```

For those unfamiliar with `go` this will create a binary `docker.test`. You can then run this binary directly

```
# ./docker.test  -test.run=TestRestartGhost .
```

Better still, we can run it under `gdb`. The `docker` image built from the Dockerfile doesn't have gdb installed by default - but it's always a quick `apt-get install gdb` away.


```
# gdb -q ./docker.test
Reading symbols from /go/src/github.com/dotcloud/docker/docker.test...done.
Loading Go Runtime support.
(gdb) break container.go:932
Breakpoint 1 at 0x468d2b: file /go/src/github.com/dotcloud/docker/container.go, line 932.
(gdb) run -test.run=TestRestartGhost .
Starting program: /go/src/github.com/dotcloud/docker/docker.test -test.run=TestRestartGhost .
warning: no loadable sections found in added symbol-file system-supplied DSO at 0x7ffff7ffd000
2013/10/27 15:01:30 Listening for HTTP on 127.0.0.1:4270 (tcp)

Breakpoint 1, github.com/dotcloud/docker.(*Container).allocateNetwork (container=0xc20016b540, ~anon0=...) at /go/src/github.com/dotcloud/docker/container.go:935
935             if !container.State.Ghost || !container.State.Running {
(gdb) p *container.runtime.networkManager.ipAllocator.network
$1 = {IP = {array = 0xc2001d93d0 "", len = 16, cap = 16}, Mask = {array = 0xc200000b18 "\377\377", len = 4, cap = 4}}
(gdb) p container.runtime.networkManager.ipAllocator.network.IP
$2 = {array = 0xc2001d93d0 "", len = 16, cap = 16}
(gdb)
```

This was really useful to me as I was then able to get a better sense of what was going on under the hood, it also allowed me to iterate on a fix once I got my failing test case.
