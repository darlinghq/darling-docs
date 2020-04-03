# Darling Shell

We plan to implement a nice and user-friendly GUI for Darling, but for now the primary way to use Darling and interact with it is via the Darling Shell.

## Basic Usage

To get a shell inside the [container](documentation/containerization), just run ''darling shell'' as a regular user. Behind the scenes, this command will start the container or connect to an already-running one and spawn a shell inside. It will also automatically load the kernel module and initialize the [prefix](Darling Prefix) contents if needed.

Inside, you'll find an emulated macOS-like environment. macOS is Unix-like, so most familiar commands will work. For example, it may be interesting to run ''ls -l /'', ''uname'' and ''sw_vers'' to explore the emulated system. Darling bundles many of the command-line tools macOS ships -- of the same ancient versions. The shell itself is Bash version 3.2.

The filesystem layout inside the container is similar to that of macOS, including the top-level ''/Applications'', ''/Users'' and ''/System'' directories. The original Linux filesystem is visible as a separate partition that's mounted on ''/Volumes/SystemRoot''. When running macOS programs under Darling, you'll likely want them to access files in you home folder; to make this very convenient, ''/Users'' and ''/home'' inside the container are both symlinked to ''/Volumes/SystemRoot/home''.

## Running Linux Binaries

You can run normal Linux binaries inside the container, too. They won't make use of Darling's libraries and [system call emulation](documentation/system_call_emulation), but they will still see the macOS-like environment:

	
	$ darling shell
	Darling [~]$ uname
	Darwin
	Darling [~]$ /Volumes/SystemRoot/bin/uname
	Linux
	Darling [~]$ /Volumes/SystemRoot/bin/ls /System
	Library


## Setting up the Environment

Darling Shell is compiled not to read ''~/.bashrc'' at startup. Instead, Darling Shell reads ''~/.dshellrc'', which allows you to configure ''PATH'' in a way that makes sense inside your prefix, for example.

If you want to alter your environment for a specific prefix only, edit ''/etc/bashrc'' inside the prefix.

## Becoming Root

Should you encounter an application that bails out because you are not root (typically because it needs write access outside your home directory), you can use the fake ''sudo'' command. It is fake, because it only makes [getuid](https///developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man2/getuid.2.html) and [geteuid](https///developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man2/geteuid.2.html) system calls return 0, but grants you no extra privileges.

## Examples


*  ''darling shell'': Opens a Bash prompt.

*  ''darling shell /usr/local/bin/someapp arg'': Execute ''/usr/local/bin/someapp'' with an argument. Note that the path is evaluated inside the [Darling Prefix](Darling Prefix). The command is started through the shell (uses ''sh -c'').

*  ''darling ~/.darling/usr/local/bin/someapp arg'': Equivalent of the previous example (which doesn't make use of the shell), assuming that the prefix is ''~/.darling''.

