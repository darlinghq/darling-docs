# Darling shell

We plan to implement a nice and user-friendly GUI for Darling, but for now the
primary way to use Darling and interact with it is via the Darling shell.

## Basic usage

To get a shell inside the [container](internals/basics/containerization.md),
just run `darling shell` as a regular user. Behind the scenes, this command will
start the container or connect to an already-running one and spawn a shell
inside. It will also automatically load the kernel module and initialize the
[prefix](darling-prefix.md) contents if needed.

Inside, you'll find an emulated macOS-like environment. macOS is Unix-like, so
most familiar commands will work. For example, it may be interesting to run `ls
-l /`, `uname` and `sw_vers` to explore the emulated system. Darling bundles
many of the command-line tools macOS ships — of the same ancient versions. The
shell itself is Bash version 3.2.

The filesystem layout inside the container is similar to that of macOS,
including the top-level `/Applications`, `/Users` and `/System` directories. The
original Linux filesystem is visible as a separate partition that's mounted on
`/Volumes/SystemRoot`. When running macOS programs under Darling, you'll likely
want them to access files in you home folder; to make this convenient, there's a
`LinuxHome` symlink in your Darling home folder that points to your Linux home
folder, as seen from inside the container; additionally, standard directories
such as `Downloads` in your Darling home folder are symlinked to the
corresponding folders in your Linux home folder.

## Running Linux Binaries

You can run normal Linux binaries inside the container, too. They won't make use
of Darling's libraries and [system call
emulation](internals/basics/system-call-emulation.md) and may not see the
macOS-like environment:

```
$ darling shell
Darling [~]$ uname
Darwin
Darling [~]$ /Volumes/SystemRoot/bin/uname
Linux
```

## Becoming root

Should you encounter an application that bails out because you are not root
(typically because it needs write access outside your home directory), you can
use the fake `sudo` command. It is fake, because it only makes
[`getuid()`](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man2/getuid.2.html)
and
[`geteuid()`](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man2/geteuid.2.html)
system calls return 0, but grants you no extra privileges.

## Examples

* `darling shell`: Opens a Bash prompt.
* `darling shell /usr/local/bin/someapp arg`: Execute `/usr/local/bin/someapp` with an argument. Note that the path is evaluated inside the [Darling Prefix](darling-prefix.md). The command is started through the shell (uses `sh -c`).
* `darling ~/.darling/usr/local/bin/someapp arg`: Equivalent of the previous example (which doesn't make use of the shell), assuming that the prefix is `~/.darling`.

