---
layout: post
title: "Debugging for Systems Engineers"
description: ""
category: 
tags: [gdb]
---
{% include JB/setup %}
Debugging for Systems Engineers
===============================

Originally posted as part of [sysadvent](http://sysadvent.blogspot.com/2014/12/day-6-debugging-for-systems-engineers.html)

# Overview

I've mentioned before that my favourite prior sysadvent articles is [Down The Rabbit Hole](http://sysadvent.blogspot.com/2010/12/day-15-down-ls-rabbit-hole.html). 

This year amongst some of the reading I've done, I really enjoyed the articles from [http://jvns.ca/](Julia Evans) on strace. I wanted to write something for sysadvent that would be interesting but focussed on debuggers. There isn't enough space here to give a full debugger tutorial but instead wanted to give some cases when I reach for a debugger and what for, with some specific tips and examples thrown in. If you're already an expert at using debuggers you can probably stop reading now.

Many of us are familiar with some of variety this quote:

> Everyone knows that debugging is twice as hard as writing a program in the first place. 
> So if you're as clever as you can be when you write it, how will you ever debug it?
> - Brian Kerninghan

learning debugging techniques for programs you haven't written is a skill I'd recommend systems engineers/administrators practise.

# Forms of debugging

## Print statement debugging

> The most effective debugging tool is still careful thought, coupled with judiciously placed print statements
> - Brian Kernighan

If you spend any time on the `golang-nuts` list you'll see that using gdb with highly concurrent runtime such as `go` is problematic, and certainly the latter part of the above quote is often recommended. If you have the luxury of being able to rebuild and rerun your application or service - such as when developing it then *print debugging* is a valuable part of your too chain. If you've written a `helloworld` program in a language you know at least one way of getting your program to print information out to you.

## Program tracing

Julia has a fantastic blog post on [debugging your programs like they're closed source](http://jvns.ca/blog/2014/04/20/debug-your-programs-like-theyre-closed-source/) covers using `strace` and friends to do black box analysis using operating systems tools.

## Using a debugger

This is where I want to share some of my knowledge. ...

### Setup

Lets build a container for our debugging experiments and launch a shell in it. I'm using Docker to make it easy for you to follow along with the same environment, lets create a directory for this with the following `Dockerfile` 

```
FROM fedora:20

RUN yum -y update && yum clean all
RUN yum -y install gdb yum-utils && yum clean all
RUN debuginfo-install -y coreutils && yum clean all
RUN mkdir -p /tmp/a && touch /tmp/a/foo
```

Most distributions ship with stripped and optimized binaries, so to get some more meaningful traces we need to install some additional packages for debug symbols. Debian and Fedora derived distros (and probably others) include ways to get these extra packages:

* [Ubuntu dbg and dbgsym package installation](https://wiki.ubuntu.com/DebuggingProgramCrash)
* [Fedora debuginfo package installation](http://fedoraproject.org/wiki/StackTraces#Installing_debuginfo_RPMs_using_yum)


```
$ docker build -t gdb .
$ docker run -ti gdb /bin/bash
```

## Running a program through the debugger

Now  off lets setup to launch a program with some args in the debugger:

```
gdb  --args /bin/ls /tmp/a
GNU gdb (GDB) Fedora 7.7.1-21.fc20
Copyright (C) 2014 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from /bin/ls...Reading symbols from /usr/lib/debug/usr/bin/ls.debug...done.
done.
(gdb)
```

As you can see `gdb` is quite verbose by default so following invocations of `gdb` will use the quiet flag `-q` to suppres the banner and license information.

All that has happened is so far is that `gdb` has loaded the symbols from the file and the additional `debuginfo`, we've not run the program yet. Lets just do that now by typing `run` - note that `(gdb) ` is the prompt output from `gdb`.

```
(gdb) run
Starting program: /usr/bin/ls /tmp/a
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".
foo
[Inferior 1 (process 28) exited normally]
Missing separate debuginfos, use: debuginfo-install pcre-8.33-7.fc20.x86_64 xz-libs-5.1.2-12alpha.fc20.x86_64
```

So far so good - we see it starts the program, prints some info on how it is running and then the `foo` is the output of our `ls command`.

To make it clearer maybe we should have chosen `ls -l` lets change the state of the debugger to update the arguments

```
(gdb) show args
Argument list to give program being debugged when it is started is "/tmp/a".
(gdb) set args -l /tmp/a
(gdb) show args
Argument list to give program being debugged when it is started is "-l /tmp/a".
(gdb) run
Starting program: /usr/bin/ls -l /tmp/a
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".
total 0
-rw-r--r-- 1 root root 0 Dec  3 15:44 foo
[Inferior 1 (process 32) exited normally]
```
 
Exit `gdb` using `quit` or `Ctrl-D`.
 
## Stop the world
 
Still in our container lets start `gdb` again this time explicitly using `ls -l`:
 
 ```
gdb -q --args /bin/ls -l /tmp/a
Reading symbols from /bin/ls...Reading symbols from /usr/lib/debug/usr/bin/ls.debug...done.
done.
(gdb)
 ```
 
Running a program straight through isn't that interesting, if you've done basic `C` you'll know that the `main` method is the entry point to our program. We're going to tell `gdb` to create a breakpoint to pause the program when it hits the main method.

```
(gdb) break main
Breakpoint 1 at 0x402c60: file src/ls.c, line 1242.
```

Using gdb you can abbreviate commands until they become ambiguous. We could have used `b main`. With the breakpoint in place lets run again:

```
Starting program: /usr/bin/ls -l /tmp/a
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".

Breakpoint 1, main (argc=3, argv=0x7fffffffe798) at src/ls.c:1242
1242	{
Missing separate debuginfos, use: debuginfo-install pcre-8.33-7.fc20.x86_64 xz-libs-5.1.2-12alpha.fc20.x86_64
(gdb) 
```

The above tells us we've hit the breakpoint at line 1242 which has a single `{` , but not much else. Lets see some more context by using the `list` command:

```
(gdb) list
1237	    }
1238	}
1239
1240	int
1241	main (int argc, char **argv)
1242	{
1243	  int i;
1244	  struct pending *thispend;
1245	  int n_files;
1246
(gdb)
```

 Repeated use of `list` will show us the next few lines, with `gdb` just hitting `return` will repeat the previous command - eg if you hit return after the above listing we see the next few lines.
 
 ```
 (gdb)
1247	  /* The signals that are trapped, and the number of such signals.  */
1248	  static int const sig[] =
1249	    {
1250	      /* This one is handled specially.  */
1251	      SIGTSTP,
1252
1253	      /* The usual suspects.  */
1254	      SIGALRM, SIGHUP, SIGINT, SIGPIPE, SIGQUIT, SIGTERM,
1255	#ifdef SIGPOLL
1256	      SIGPOLL,
```

Even though we're paging through the source code we've still not executed it - if you get confused you can always ask `gdb` where you are:

```
(gdb) where
#0  main (argc=3, argv=0x7fffffffe798) at src/ls.c:1242
```

If you're not familiar with `C` the `argc` is the count of arguments and `argv` is an array of strings. We can use `gdb`'s `print` command - which abbreviates to `p` to inspect these.

```
(gdb) print argc
$2 = 3
(gdb) p argv[0]
$3 = 0x7fffffffe97f "/usr/bin/ls"
(gdb) p argv[1]
$4 = 0x7fffffffe98b "-l"
(gdb) p argv[2]
$5 = 0x7fffffffe98e "/tmp/a"
```

For now ignore the `$2 =` part and just focus on the values. For `argc` which is an `int` we just get the numeric value. Using the size we can look at the three strings forming `argv` using array access.

`ls` is quite a large program - for now lets tell it to continue executing:

```
(gdb) cont
Continuing.
total 0
-rw-r--r-- 1 root root 0 Dec  3 15:44 foo
[Inferior 1 (process 50) exited normally]
```

Now we know a little on how to set a breakpoint and get some basic information on variables in the program. Lets try another program - setting a breakpoint in another method. For now I'm choosing to use `cat`.

```
[root@7dffa749b76c /]# gdb -q --args /bin/cat /etc/passwd
Reading symbols from /bin/cat...Reading symbols from /usr/lib/debug/usr/bin/cat.debug...done.
done.
(gdb) break simple_cat
Breakpoint 1 at 0x4026b8: file src/cat.c, line 177.
(gdb) run
Starting program: /usr/bin/cat /etc/passwd

Breakpoint 1, main (argc=2, argv=<optimized out>) at src/cat.c:730
730	          ok &= simple_cat (ptr_align (inbuf, page_size), insize);
```

So we stop before we execute the simple_cat method

```
(gdb) list
725	             || show_tabs || squeeze_blank))
726	        {
727	          insize = MAX (insize, outsize);
728	          inbuf = xmalloc (insize + page_size - 1);
729
730	          ok &= simple_cat (ptr_align (inbuf, page_size), insize);
731	        }
732	      else
733	        {
734	          inbuf = xmalloc (insize + 1 + page_size - 1);
```

Rather than continuing the exection we are going to tell the debugger to execute the simple_cat method then stop using the `next` command. Some debuggers call this "stepping over". 

```
(gdb) next
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
games:x:12:100:games:/usr/games:/sbin/nologin
ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody:x:99:99:Nobody:/:/sbin/nologin
dbus:x:81:81:System message bus:/:/sbin/nologin
Breakpoint 1, main (argc=2, argv=<optimized out>) at src/cat.c:730
730	          ok &= simple_cat (ptr_align (inbuf, page_size), insize);
(gdb) p ok
$1 = true
```

As you can see we've not moved lines but have executed the subroutine - which output the file and returned. We can see the value of `ok` set. Stepping over is useful when you want to follow the flow of something but don't care about what it is calling.

Type `cont` to continue execution.

## Going deeper

In addition to stepping over, most debuggers allow you to follow the code into the method by stepping into. In `gdb` the command is `step`

```
gdb -q --args /bin/cat /etc/passwd
Reading symbols from /bin/cat...Reading symbols from /usr/lib/debug/usr/bin/cat.debug...done.
done.
(gdb) break simple_cat
Breakpoint 1 at 0x4026b8: file src/cat.c, line 177.
(gdb) run
Starting program: /usr/bin/cat /etc/passwd

Breakpoint 1, main (argc=2, argv=<optimized out>) at src/cat.c:730
730	          ok &= simple_cat (ptr_align (inbuf, page_size), insize);
(gdb) step
simple_cat (bufsize=65536,
    buf=0x60f000 "root:x:0:0:root:/root:/bin/bash\nbin:x:1:1:bin:/bin:/sbin/nologin\ndaemon:x:2:2:daemon:/sbin:/sbin/nologin\nadm:x:3:4:adm:/var/adm:/sbin/nologin\nlp:x:4:7:lp:/var/spool/lpd:/sbin/nologin\nsync:x:5:0:sync:/"...) at src/cat.c:177
177	      if (n_read == 0)
```
When you've entered a function `cont` acts as a sort of step return where you run all the code in the method and stop at the return point.

```
(gdb) cont
Continuing.
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
games:x:12:100:games:/usr/games:/sbin/nologin
ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody:x:99:99:Nobody:/:/sbin/nologin
dbus:x:81:81:System message bus:/:/sbin/nologin

Breakpoint 1, main (argc=2, argv=<optimized out>) at src/cat.c:730
730	          ok &= simple_cat (ptr_align (inbuf, page_size), insize);
```



## Getting to know your codebases

I learned this technique from a friend as an early thing I do when learning a new service or application I'm going to work on or support.

Requirements:

1. I make sure have a US Letter/A4 pad which I usually make sure is landscape, some pencils/pens next to me.
2. Next step is to learn how to checkout, build and run the application locally.
3. Choose a point to dive in for - eg for a command line app this might be `main()` 

Then I use step-debugging to follow through and sketch out the main conceptually flow of the program, and dive in to parts I'm interested in. This works particularly well within IDE debuggers with the source code such as Eclipse or Intellij (or gud mode of emacs). Where you can set a break from the sourcecode and using the debugger. If you're working in a Java environment I'd strongly encourage you to learn the IDE your developer's use and how the debugger works there. Learning to navigate the codebase and understanding what live thread and stack dumps look like can really help you reason better about the application when troubleshooting.

Test-driven debugging
---------------------

This is an expansion of the previous point - some language debuggers such as `pdb` or `pry` make it easy to add a breakpoint programatically within the test, or using the IDE or even just running the test in the debugger set the breakpoint and step into.

I've often used this as a way to improve my understanding of the code under test and the API before writing a new test. It allows you to drill deep.

Debugging production issues
---------------------------

At various points in my career I've had to dive in at quite a low level and figure out what is happening - with  large and complex multithreaded applications this can be non-trivial. Often you really just want to get the stack trace of the program when the issue manifests.

You want to apply careful thought before ever doing this.

* Come up with a hypothesis
* Attempt to isolate problem
* Run with debugger in isolated environment

 file descriptor leaks within a large JVM application including JNI database client.

When facing these issues `gdb` can be very powerful - in addition to the example of running the program in `gdb` you can attach to processes or analyse core dumps.

# Scripting your debugger

`gdb` supports various forms of automation and scripting - for example you can define helper functions. 
One reason I've used this in the path is when debugging the JVM - the JVM users various [signals](http://www.oracle.com/technetwork/java/javase/signals-139944.html#gbzbl) internally so we can to configure gdb to ignore these. 

```
define java_signals
  handle SIGTSTP noprint nostop
  handle SIGSEGV noprint nostop
  handle SIGQUIT noprint nostop
  handle SIGABRT stop print nopass 
end
```

`SIGQUIT` gives us a JVM stacktrace so we can understand how where we are natively corresponds to where we are in our Java codebase. You can also define a `gdb` function to force that.

```
define jtrace
  call kill(getpid(), 3)
  cont
end
```

The times I've had to connect debugger to real live production tasks - this should always be done with caution - if you can ensure it's not serving real user traffic either by  is best as debuggers suspend execution which can impact user requests. You also may find that your monitoring or health checking may fail and kill the process under the debugger. Often a stacktrace is enough.

`gdb` supports attaching commands on a breakpoint so you can run some code so you could for example run the above `jtrace` function to force a Java stack trace when you hit some specific state and continue. If you're absolutely stuck with no alternative but to run the debugger on a production serving process practice in a test environment first to learn how to use `breakpoint commands` to minimize the amount of time the program will be suspended.

In addition to gdb scripting modern `gdb` supports python to write richer extensions, pretty printers, etc. Two examples I've used as reference are [gdb-heap](https://fedorahosted.org/gdb-heap/) which allows inspection of python's memory usage and `Go`'s [runtime-gdb.py](https://golang.org/src/pkg/runtime/runtime-gdb.py) which is an interesting way to learn some of the runtime internals by seeing how it looks up goroutines, etc. 

## Conclusion

So we've learned the basic debugger primatives of

* Setting a breakpoint
* Inspecting some state
* Advancing execution by stepping over a call
* Debugging further in by stepping into a call

We've taken a look at some actual real world tasks where debugger knowledge is useful. 

Working with multithreaded, concurrent and complex applications with a debugger is more challenging than the simple walk through here. Like many tools in your portfolio it's best to practice using your debuggers before you're in a situation where that's the only way you could get information.

Further Reading
---------------

* [GDB Documentation](http://www.gnu.org/software/gdb/documentation/)
* [lldb to GDB reference](http://lldb.llvm.org/lldb-gdb.html)
* [Julia Evans on strace](http://jvns.ca/blog/categories/strace/)
* [Tom Tromey's GDB blog posts](http://tromey.com/blog/?cat=17)

