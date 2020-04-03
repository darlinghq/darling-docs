# What to Try

Here are some things you may want to try after installing Darling.

## Print "Hello World"

See if Darling can print the famous greeting:

	
	Darling [~]$ echo Hello World
	Hello World


It works!

## Run uname

''uname'' is a standard Unix command to get the name (and optionally the version) of the core OS. On Linux distributions, it prints "Linux":

	
	$ uname
	Linux


But Darling emulates a complete Darwin environment, so running ''uname'' results in "Darwin":

	
	Darling [~]$ uname
	Darwin


## Run sw_vers

''sw_vers'' (for "software version") is a Darwin command that prints the user-facing name, version and code name (such as "El Capitan") of the OS:

	
	Darling [~]$ sw_vers
	ProductName:    Mac OS X
	ProductVersion: 10.12
	BuildVersion:   Darling


## Explore the file system

Explore the file system Darling presents to Darwin programs, e.g.:

	
	Darling [~]$ ls -l /
	`<...>`
	Darling [~]$ ls /System/Library
	`<...>`
	Darling [~]$ ls /usr/lib
	`<...>`
	Darling [~]$ ls -l /Volumes
	`<...>`


## Inspect the Mach-O binaries

Darling ships with tools like ''nm'' and ''otool'' that let you inspect Mach-O binaries, ones that make up Darling and any third-party ones:

	
	Darling [~]$ nm /usr/lib/libobjc.A.dylib
	`<...>`
	Darling [~]$ otool -l /bin/bash
	`<...>`


## Explore process memory layout

While Darling emulates a complete Darwin system, it's still powered by Linux underneath. Sometimes,
this may prove useful. For example, you can use Linux's ''/proc'' pseudo-filesystem to explore the
running processes. Let's use ''cat'' to explore its own process memory layout:

	
	Darling [~]$ cat /proc/self/maps
	`<...>`


## Check out the mounts

Darling runs in a mount namespace that's separate from the host. You can use host's native ''mount'' tool to inspect it:

	
	Darling [~]$ /Volumes/SystemRoot/usr/bin/mount | column -t
	/Volumes/SystemRoot/dev/sda3  on  /Volumes/SystemRoot  type  ext4     (rw,relatime,seclabel)
	overlay                       on  /                    type  overlay  (rw,relatime,seclabel,lowerdir=/usr/local/libexec/darling,upperdir=/home/user/.darling,workdir=/home/user/.darling.workdir)
	proc                          on  /proc                type  proc     (rw,relatime)
	`<...>`


Notice that not only can you simply run a native ELF executable installed on the host, you can also pipe its output directly into a Darwin command (like ''column'' in this case).

Alternatively, you can read the same info from the ''/proc'' pseudo-filesystem:

	
	Darling [~]$ column -t /proc/self/mounts
	`<...>`


## List running processes

Darling emulates the BSD ''sysctl''s that are needed for ''ps'' to work:

	
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


## Read the manual

Darling ships with many ''man'' pages you can read:

	
	Darling [~]$ man dyld


## Run a script

Like Darwin, Darling ships with a build of Python, Ruby and Perl. You can try running a script or exploring them interactively.

	
	Darling [~]$ python
	Python 2.7.10 (default, Sep  8 2018, 13:32:07) 
	[GCC 4.2.1 Compatible Clang 6.0.1 (tags/RELEASE_601/final)] on darwin
	Type "help", "copyright", "credits" or "license" for more information.
	>>> import sys
	>>> sys.platform
	'darwin'



## Trace a process

Use our ''xtrace'' tool to trace the emulated Darwin syscalls a process makes:

	
	Darling [~]$ xtrace arch
	`<...>`
	[223] mach_timebase_info_trap (...)
	[223] mach_timebase_info_trap () -> KERN_SUCCESS
	[223] issetugid (...)
	[223] issetugid () -> 0
	[223] host_self_trap ()
	[223] host_self_trap () -> port right 2563
	[223] mach_msg_trap (...)
	[223] mach_msg_trap () -> KERN_SUCCESS
	[223] _kernelrpc_mach_port_deallocate_trap (task=2563, name=-6)
	[223] _kernelrpc_mach_port_deallocate_trap () -> KERN_SUCCESS
	[223] ioctl (...)
	[223] ioctl () -> 0
	[223] fstat64 (...)
	[223] fstat64 () -> 0
	[223] ioctl (...)
	[223] ioctl () -> 0
	[223] write_nocancel (...)
	i386
	[223] write_nocancel () -> 5
	[223] exit (...)


## Control running services

Use ''launchctl'' tool to control ''launchd'':

	
	Darling [~]$ launchctl list
	PID	Status	Label
	55	-	0x7ffb0dc015f0.anonymous.launchctl
	8	-	0x7ffb0dc04ef0.anonymous.shellspawn
	9	-	0x7ffb0dc04c70.anonymous.bash
	6	-	org.darlinghq.shellspawn
	10	-	com.apple.notifyd
	-	0	com.apple.periodic-daily
	-	0	com.apple.periodic-monthly
	-	0	com.apple.newsyslog
	-	0	com.apple.periodic-weekly
	7	-	com.apple.syslogd
	-	0	com.apple.var-db-dslocal-backup
	-	-11	com.apple.aslmanager
	-	0	com.apple.launchctl.System


Read ''man launchctl'' for more information of other commands ''launchctl'' has.

## Fetch a webpage

See if networking works as it should:

	
	Darling [~]$ curl http://darlinghq.org
	
	`<!DOCTYPE html>`
	
	`<html lang="en-US">`
	`<head>`
	`<...>`


## Try using sudo

Just like [ the real Mac OS X may](https///www.macrumors.com/2017/11/28/macos-high-sierra-bug-admin-access/ ), Darling allows you to get root priveleges without having to enter any password, except in our case it's a feature:

	
	Darling [~]$ whoami
	user
	Darling [~]$ sudo whoami
	root


Of course, our ''sudo'' command only gives the program the *impression* it's running as root; in reality, it still runs with privileges of your user. Some programs explicitly check that they're running as root, so you can use our ''sudo'' to convince them to run.

## Use a package manager

Download and install the Rudix Package Manager:

Note: Not currently working due to lack of TLS support.

	
	Darling [~]$ curl -s https://raw.githubusercontent.com/rudix-mac/rpm/2015.10.20/rudix.py | sudo python - install rudix


Now you can install arbitrary packages using the ''rudix'' command:

	
	Darling [~]$ sudo rudix install wget mc


## Try running Midnight Commander

If you've installed Midnight Commander (''mc'' package in Rudix), launch it to see if it runs smoothly:

	
	Darling [~]$ mc


## Manually install a package

You can also try installing a ''.pkg'' file manually using the ''installer'' command:

	
	Darling [~]$ installer -pkg mc-4.8.7-0.pkg -target /


Unlike macOS, Darling also ships with an ''uninstaller'' command which you can use to easily uninstall packages.

## Attach disk images

Darling ships with an implementation of ''hdiutil'', a tool that allows you to attach and detach DMG disk images:

	
	Darling [~]$ hdiutil attach Downloads/SomeApp.dmg
	/Volumes/SomeApp
	Darling [~]$ ls /Volumes/SomeApp
	`<...>`
	Darling [~]$ cp -r /Volumes/SomeApp/SomeApp.app /Applications/
	Darling [~]$ hdiutil detach /Volumes/SomeApp


## Run neofetch

Get the ''neofetch.sh'' script from [ its homepage](https///github.com/dylanaraps/neofetch ) and run it:

	
	Darling [~]$ bash neofetch.sh
	                    'c.          user@User-VirtualBox
	                 ,xNMM.          ------------------------
	               .OMMMMo           OS: macOS Sierra 10.12 Darling x86_64
	               OMMM0,            Kernel: 16.0.0
	     .;loddo:' loolloddol;.      Uptime: 3 hours, 20 mins
	   cKMMMMMMMMMMNWMMMMMMMMMM0:    Shell: bash 3.2.57
	 .KMMMMMMMMMMMMMMMMMMMMMMMWd.    DE: Aqua
	 XMMMMMMMMMMMMMMMMMMMMMMMX.      WM: Quartz Compositor
	;MMMMMMMMMMMMMMMMMMMMMMMM:       Terminal: /dev/pts/0
	:MMMMMMMMMMMMMMMMMMMMMMMM:       CPU: GenuineIntel
	.MMMMMMMMMMMMMMMMMMMMMMMMX.      Memory: 0MiB / 983MiB
	 kMMMMMMMMMMMMMMMMMMMMMMMMWd.
	 .XMMMMMMMMMMMMMMMMMMMMMMMMMMk
	  .XMMMMMMMMMMMMMMMMMMMMMMMMK.
	    kMMMMMMMMMMMMMMMMMMMMMMd
	     ;KMMMMMMMWXXWMMMMMMMk.
	       .cooc,.    .,coo:.
	


''neofetch'' exercises a lot of the Darwin internals, including BSD ''sysctl'' calls, Mach IPC and host info API, and the "defaults" subsystem (accessed by the ''defaults'' tool, which is implemented in Objective-C on top of Foundation's ''NSUserDefaults'').
## Compile and run a program

If you have Xcode installed, you can use the SDK it provides to compile and run a program. First, select an SDK using our implementation of the ''xcode-select'' tool:

	
	Darling [~]$ xcode-select --switch /Applications/Xcode.app


Now, build a "Hello World" C program using the Clang compiler:

	
	Darling [~]$ cat > helloworld.c
	#include `<stdio.h>`
	
	int main() {
	    puts("Hello World!");
	}
	Darling [~]$ clang helloworld.c -o helloworld


And run it:

	
	Darling [~]$ ./helloworld
	Hello world!


The whole compiler stack works, how cool is that! Now, let's try Swift:

	
	Darling [~]$ cat > hi.swift
	print("Hi!")
	Darling [~]$ swiftc hi.swift
	Darling [~]$ ./hi
	Hi!

