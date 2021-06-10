# What to try

Here are some things you may want to try after installing Darling.

## Print "Hello World"

See if Darling can print the famous greeting:

```
Darling [~]$ echo Hello World
Hello World
```

It works!

## Run uname

`uname` is a standard Unix command to get the name (and optionally the version)
of the core OS. On Linux distributions, it prints "Linux":

```
$ uname
Linux
```

But Darling emulates a complete Darwin environment, so running `uname` results in "Darwin":

```	
Darling [~]$ uname
Darwin
```

## Run sw_vers

`sw_vers` (for "software version") is a Darwin command that prints the
user-facing name, version and code name (such as "El Capitan") of the OS:

```
Darling [~]$ sw_vers
ProductName:    Mac OS X
ProductVersion: 10.14
BuildVersion:   Darling
```

## Explore the file system

Explore the file system Darling presents to Darwin programs, e.g.:

```
Darling [~]$ ls -l /
...
Darling [~]$ ls /System/Library
Caches Frameworks OpenSSL ...
Darling [~]$ ls /usr/lib
...
Darling [~]$ ls -l /Volumes
...
```

## Inspect the Mach-O binaries

Darling ships with tools like `nm` and `otool` that let you inspect Mach-O
binaries, ones that make up Darling and any third-party ones:

```
Darling [~]$ nm /usr/lib/libobjc.A.dylib
...
Darling [~]$ otool -L /bin/bash
/bin/bash:
	/usr/lib/libncurses.5.4.dylib (compatibility version 5.4.0, current version 5.4.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1238.0.0)
```

## Explore process memory layout

While Darling emulates a complete Darwin system, it's still powered by Linux underneath. Sometimes,
this may prove useful. For example, you can use Linux's `/proc` pseudo-filesystem to explore the
running processes. Let's use `cat` to explore its own process memory layout:

```
Darling [~]$ cat /proc/self/maps
...
7ffff7ffb000-7ffff7ffd000 rwxp 00000000 fe:01 20482                      /home/user/.darling/bin/cat
7ffff7ffd000-7ffff7ffe000 rw-p 00002000 fe:01 20482                      /home/user/.darling/bin/cat
7ffff7ffe000-7ffff7fff000 r--p 00003000 fe:01 20482                      /home/user/.darling/bin/cat
7ffff7fff000-7ffff80f3000 rwxp 00182000 fe:01 60690                      /home/user/.darling/usr/lib/dyld
7ffff80f3000-7ffff80fc000 rw-p 00276000 fe:01 60690                      /home/user/.darling/usr/lib/dyld
7ffff80fc000-7ffff8136000 rw-p 00000000 00:00 0
7ffff8136000-7ffff81d6000 r--p 0027f000 fe:01 60690                      /home/user/.darling/usr/lib/dyld
7fffffdde000-7fffffdff000 rw-p 00000000 00:00 0                          [stack]
7fffffe00000-7fffffe01000 r--s 00000000 00:0e 8761                       anon_inode:[commpage]
```

## Check out the mounts

Darling runs in a mount namespace that's separate from the host. You can use
host's native `mount` tool to inspect it:

```
Darling [~]$ /Volumes/SystemRoot/usr/bin/mount | column -t
/Volumes/SystemRoot/dev/sda3  on  /Volumes/SystemRoot  type  ext4     (rw,relatime,seclabel)
overlay                       on  /                    type  overlay  (rw,relatime,seclabel,lowerdir=/usr/local/libexec/darling,upperdir=/home/user/.darling,workdir=/home/user/.darling.workdir)
proc                          on  /proc                type  proc     (rw,relatime)
<...>
```

Notice that not only can you simply run a native ELF executable installed on the
host, you can also pipe its output directly into a Darwin command (like `column`
in this case).

Alternatively, you can read the same info from the `/proc` pseudo-filesystem:

```
Darling [~]$ column -t /proc/self/mounts
<...>
```

## List running processes

Darling emulates the BSD `sysctl`s that are needed for `ps` to work:

```
Darling [~]$ ps aux
USER   PID  %CPU %MEM      VSZ    RSS   TT  STAT STARTED      TIME COMMAND
user    32   0.0  0.4  4229972  13016   ??  ?     1Jan70   0:00.05 ps aux
user     5   0.0  0.5  4239500  15536   ??  ?     1Jan70   0:00.22 /bin/launchctl bootstrap -S System
user     6   0.0  0.4  4229916  11504   ??  ?     1Jan70   0:00.09 /usr/libexec/shellspawn
user     7   0.0  0.6  4565228  17308   ??  ?     1Jan70   0:00.14 /usr/sbin/syslogd
user     8   0.0  0.6  4407876  18936   ??  ?     1Jan70   0:00.15 /usr/sbin/notifyd
user    29   0.0  0.2  4229948   7584   ??  ?N    1Jan70   0:00.03 /usr/libexec/shellspawn
user    30   0.0  0.5  4231736  14268   ??  ?     1Jan70   0:00.11 /bin/bash
user     1   0.0  0.5  4256056  15484   ??  ?     1Jan70   0:00.25 launchd
```

## Read the manual

Darling ships with many `man` pages you can read:

```
Darling [~]$ man dyld
```

## Run a script

Like Darwin, Darling ships with a build of Python, Ruby and Perl. You can try
running a script or exploring them interactively.

```
Darling [~]$ python
Python 2.7.10 (default, Sep  8 2018, 13:32:07) 
[GCC 4.2.1 Compatible Clang 6.0.1 (tags/RELEASE_601/final)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import sys
>>> sys.platform
'darwin'
```


## Trace a process

Use our `xtrace` tool to trace the emulated Darwin syscalls a process makes:

```
Darling [~]$ arch
i386
Darling [~]$ xtrace arch
<...>
[321] mach_timebase_info_trap() -> numer = 1, denom = 1
[321] issetugid() -> 0
[321] host_self_trap() -> port right 2563
[321] mach_msg_trap(0x7fffffdff270, MACH_SEND_MSG|MACH_RCV_MSG, 40, 320, port 1543, 0, port 0)
[321]         {remote = copy send 2563, local = make send-once 1543, id = 200}, 16 bytes of inline data
[321]         mach_host::host_info(copy send 2563, 1, 12)
[321] mach_msg_trap() -> KERN_SUCCESS
[321]         {local = move send-once 1543, id = 300}, 64 bytes of inline data
[321]         mach_host::host_info() -> [8, 8, 0, 0, 3104465855, 4160607681, 4160604077, 0, 4292867120, 4292867064, 4151031935, 3160657432], 12 
[321] _kernelrpc_mach_port_deallocate_trap(task 259, port name 2563) -> KERN_SUCCESS
[321] ioctl(0, 1074030202, 0x7fffffdff3d4) -> 0
[321] fstat64(1, 0x7fffffdfef80) -> 0
[321] ioctl(1, 1074030202, 0x7fffffdfefd4) -> 0
[321] write_nocancel(1, 0x7f823680a400, 5)i386
 -> 5
[321] exit(0)
```

## Control running services

Use `launchctl` tool to control `launchd`:

```
Darling [~]$ launchctl list
PID	Status	Label
323	-	0x7ffea3407da0.anonymous.launchctl
49	-	0x7ffea3406d50.anonymous.shellspawn
50	-	0x7ffea3406a20.anonymous.bash
39	-	0x7ffea3406350.anonymous.shellspawn
40	-	0x7ffea3405fd0.anonymous.bash
-	0	com.apple.periodic-monthly
-	0	com.apple.var-db-dslocal-backup
31	-	com.apple.aslmanager
-	0	com.apple.newsyslog
-	0	com.apple.periodic-daily
-	0	com.apple.securityd
19	-	com.apple.memberd
23	-	com.apple.notifyd
20	-	org.darlinghq.iokitd
-	0	com.apple.periodic-weekly
21	-	org.darlinghq.shellspawn
22	-	com.apple.syslogd
-	0	com.apple.launchctl.System

Darling [~]$ sudo launchctl bstree
System/
    A  org.darlinghq.iokitd
    A  com.apple.system.logger
    D  com.apple.activity_tracing.cache-delete
    D  com.apple.SecurityServer
    A  com.apple.aslmanager
    A  com.apple.system.notification_center
    A  com.apple.PowerManagement.control
    com.apple.xpc.system (XPC Singleton Domain)/
```

Read `man launchctl` for more information of other commands `launchctl` has.

## Fetch a webpage

See if networking works as it should:

```
Darling [~]$ curl http://example.og
<!doctype html>
<html>
<head>
    <title>Example Domain</title>
<...>
```

## Try using sudo

Just like [the real Mac OS X
may](https://www.macrumors.com/2017/11/28/macos-high-sierra-bug-admin-access/),
Darling allows you to get root privileges without having to enter any password,
except in our case it's a feature:

```	
Darling [~]$ whoami
user
Darling [~]$ sudo whoami
root
```

Of course, our `sudo` command only gives the program the *impression* it's
running as root; in reality, it still runs with privileges of your user. Some
programs explicitly check that they're running as root, so you can use our
`sudo` to convince them to run.

## Use a package manager

Download and install the Rudix Package Manager:

Note: Not currently working due to lack of TLS support.

```	
Darling [~]$ curl -s https://raw.githubusercontent.com/rudix-mac/rpm/2015.10.20/rudix.py | sudo python - install rudix
```

Now you can install arbitrary packages using the `rudix` command:

```
Darling [~]$ sudo rudix install wget mc
```

### Install Homebrew

macOS's de-facto package manager, [Homebrew](https://brew.sh/), installs and works under Darling (albeit with issues with certain formulas).

```
Darling [~]$ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Now you can install packages just like you would on a real macOS installation:

```
Darling [~]$ brew install wget
```

## Try running Midnight Commander

If you've installed Midnight Commander (`mc` package in Rudix), launch it to see
if it runs smoothly:

```
Darling [~]$ mc
```

## Manually install a package

You can also try installing a `.pkg` file manually using the `installer`
command:

```
Darling [~]$ installer -pkg mc-4.8.7-0.pkg -target /
```

Unlike macOS, Darling also ships with an `uninstaller` command which you can use
to easily uninstall packages.

## Attach disk images

Darling ships with an implementation of `hdiutil`, a tool that allows you to
attach and detach DMG disk images:

```
Darling [~]$ hdiutil attach Downloads/SomeApp.dmg
/Volumes/SomeApp
Darling [~]$ ls /Volumes/SomeApp
SomeApp.app <...>
Darling [~]$ cp -r /Volumes/SomeApp/SomeApp.app /Applications/
Darling [~]$ hdiutil detach /Volumes/SomeApp
```

## Check out user defaults

macOS software uses the so-called "user defaults" system to store preferences.
You can access it using the `defaults` tool:

```
Darling [~]$ defaults read
```

For more information about using `defaults`, read the manual:

```
Darling [~]$ man defaults
```

## Run neofetch

Get the `neofetch.sh` script from [its homepage](https://github.com/dylanaraps/neofetch ) and run it:

```
Darling [~]$ bash neofetch.sh
                    'c.          user@hostname
                 ,xNMM.          ------------------------
               .OMMMMo           OS: macOS Mojave 10.14 Darling x86_64
               OMMM0,            Kernel: 16.0.0
     .;loddo:' loolloddol;.      Uptime: 1 day, 23 hours, 25 mins
   cKMMMMMMMMMMNWMMMMMMMMMM0:    Shell: bash 3.2.57
 .KMMMMMMMMMMMMMMMMMMMMMMMWd.    DE: Aqua
 XMMMMMMMMMMMMMMMMMMMMMMMX.      WM: Quartz Compositor
;MMMMMMMMMMMMMMMMMMMMMMMM:       WM Theme: Blue (Print: Entry, AppleInterfaceStyle, Does Not Exist)
:MMMMMMMMMMMMMMMMMMMMMMMM:       Terminal: /dev/pts/1
.MMMMMMMMMMMMMMMMMMMMMMMMX.      CPU: GenuineIntel
 kMMMMMMMMMMMMMMMMMMMMMMMMWd.    Memory: 12622017MiB / 2004MiB
 .XMMMMMMMMMMMMMMMMMMMMMMMMMMk
  .XMMMMMMMMMMMMMMMMMMMMMMMMK.
    kMMMMMMMMMMMMMMMMMMMMMMd
     ;KMMMMMMMWXXWMMMMMMMk.
       .cooc,.    .,coo:.
```

`neofetch` exercises a lot of the Darwin internals, including BSD `sysctl`
calls, Mach IPC and host info API, and the "user defaults" subsystem.

## Compile and run a program

If you have [Xcode SDK installed](installing-software.md#command-line-developer-tools),
you can compile and run programs.

```
Darling [~]$ xcode-select --switch /Applications/Xcode.app
```

Now, build a "Hello World" C program using the Clang compiler:

```
Darling [~]$ cat > hello-world.c
#include <stdio.h>

int main() {
    puts("Hello World!");
}
Darling [~]$ clang hello-world.c -o hello-world
```

And run it:

```
Darling [~]$ ./hello-world
Hello world!
```

The whole compiler stack works, how cool is that! Now, let's try Swift:

```
Darling [~]$ cat > hi.swift
print("Hi!")
Darling [~]$ swiftc hi.swift
Darling [~]$ ./hi
Hi!
```

## Try running apps
Darling has experimental support for graphical applications written using Cocoa.
If you have a simple app installed, you can try running it:

```
Darling [~]$ /Applications/HelloWorld.app/Contents/MacOS/HelloWorld
```
