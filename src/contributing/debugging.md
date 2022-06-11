# Debugging

We provide a [build of
LLDB](https://darling-misc.s3.eu-central-1.amazonaws.com/lldb.tar.bz2) that is known
to work under Darling. It is built from vanilla sources, i.e. without any
modifications.

That doesn't mean it works *reliably*. Fully supporting a debugger is a complex
task, so LLDB is known to be buggy under Darling.

## Troubleshooting LLDB

If you want to troubleshoot a problem with how LLDB runs under Darling, you need
to make `debugserver` produce a log file.

If running `debugserver` separately, add `-l targetfile`. If using LLDB directly
(which spawns `debugserver` as a subprocess automatically), pass an additional
environment variable to LLDB:

```
LLDB_DEBUGSERVER_LOG_FILE=somelogfile.txt
```

## External `DebugServer`

If you're having trouble using LLDB normally, you may get luckier by running the
`debugserver` separately.

In one terminal, start the `debugserver`:

```
$ ./debugserver 127.0.0.1:12345 /bin/bash
```

In another terminal, connect to the server in LLDB:

```	
$ ./lldb
(lldb) platform select remote-macosx
Platform: remote-macosx
Connected: no
(lldb) process connect connect://127.0.0.1:12345
```

Please note that environment variables may be missing by default, if used like this.

## Debug with core dump

If you are unable to start your application through lldb, you can generate a core dump and load it through lldb. You will need to enable some options on the Linux side before you are able to generate a core dump. You will to tell Linux that you want to generate a core dump, and that there is no size limit for the core dump.

```
sudo sysctl -w kernel.core_pattern=core_dump
ulimit -c unlimited
```

Note that the core dump will be stored into the current working directory that Linux (not Darling) is pointing to. So you should `cd` into the directory you want the core dump to be stored in before you execute `darling shell`. From there, you can execute the application.

```
cd /path/you/want/to/store/core/dump/in
darling shell
/path/to/application/executable
```

If everything was set up properly, you should find a file called `core_dump`. It will be located in the current working directory that Linux is pointing to. Once you found the file, you can load it up on lldb.

```
lldb --core /path/to/core/dump/file
```

## Built-in debugging utilities

### malloc

**libmalloc** (the default library that implements `malloc()`, `free()` and
friends) supports many debug features that can be turned on via environment,
among them:

* `MallocScribble` (fill freed memory with `0x55`)
* `MallocPreScribble` (fill allocated memory with `0xaa`)
* `MallocGuardEdges` (guard edges of large allocations)

When this is not enough, you can use **libgmalloc**, which is (for the most
part) a drop-in replacement for libmalloc. This is how you use it:

```
$ DYLD_INSERT_LIBRARIES=/usr/lib/libgmalloc.dylib DYLD_FORCE_FLAT_NAMESPACE=1 ./test`
```

libgmalloc catches memory issues such as use-after-free and buffer overflows,
and attempts to do this the moment they occur, rather than wait for them to mess
up some internal state and manifest elsewhere. It does that by placing each
allocation on its own memory page(s), and adding a guard page next to it (like
`MallocGuardEdges` on steroids) — by default, after the allocated buffer (to
catch buffer overruns), or with `MALLOC_PROTECT_BEFORE`, before it (to catch
buffer underruns). You're likely want to try it both ways, with and without
`MALLOC_PROTECT_BEFORE`. Another useful option is `MALLOC_FILL_SPACE` (similar
to `MallocPreScribble` from above). See
[libgmalloc(3)](https://www.manpagez.com/man/3/libgmalloc/) for more details.

### Objective-C and Cocoa

In Objective-C land, you can set `NSZombieEnabled=YES` in order to detect
use-after-free of Objective-C objects. This replaces `-[NSObject dealloc]` with
a different implementation that does not deallocate the memory, but instead
turns the object into a "zombie". Upon receiving any message (which indicates a
use-after-free), the zombie will log the message and abort the process.

Another useful option is `NSObjCMessageLoggingEnabled=YES`, which will instruct
the Objective-C runtime to log all the sent messages to a file in `/tmp`.

In AppKit, you can set the `NSShowAllViews` default (e.g. with `-NSShowAllViews
1` on the command line) to cause it to draw a colorful border around each view.

You can find more tips (not all of which work under Darling)
[here](https://developer.apple.com/library/archive/technotes/tn2124/_index.html).

### xtrace

You can use **xtrace** to trace Darwin syscalls a program makes, a lot like
using `strace` on Linux:

```
$ xtrace vm_stat
[139] fstat64(1, 0x7fffffdfe340) -> 0
[139] host_self_trap() -> port right 2563
[139] mach_msg_trap(0x7fffffdfdfc0, MACH_SEND_MSG|MACH_RCV_MSG, 40, 1072, port 1543, 0, port 0)
[139]         {remote = copy send 2563, local = make send-once 1543, id = 219}, 16 bytes of inline data
[139]         mach_host::host_statistics64(copy send 2563, 4, 38)
[139]     mach_msg_trap() -> KERN_SUCCESS
[139]         {local = move send-once 1543, id = 319}, 168 bytes of inline data
[139]         mach_host::host_statistics64() -> [75212, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1024, 4159955152, 720014745, 503911965, 0, 4160737656, 2563, 4292863152, 4160733284, 4160733071, 0], 38 
[139] write_nocancel(1, 0x7fd456800600, 58)Mach Virtual Memory Statistics: (page size of 4096 bytes)
-> 58
[139] write_nocancel(1, 0x7fd456800600, 49)Pages free:                               75212.
-> 49
...
```

`xtrace` can trace both BSD syscalls and Mach traps. For `mach_msg()` in
particular, `xtrace` additionally displays the message being sent or received,
as well as tries to decode it as a MIG routine call or reply.

Note that `xtrace` only traces emulated Darwin syscalls, so any native Linux
syscalls made (usually by native ELF libraries) will not be displayed, which
means information and open file descriptors may appear to come from nowhere in
those cases.
