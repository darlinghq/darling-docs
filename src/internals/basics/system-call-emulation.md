# System call emulation

System calls are the most critical interface of all user space software. They
are the means of invoking kernel's functionality. Without them, software would
not be able to communicate with the user, access the file system or establish
network connections.

Every kernel provides a very specific set of system calls, even if there is a
similar set of libc APIs available on different systems. For example, the system
call set greatly differs between Linux and FreeBSD.

MacOS uses a kernel named XNU. XNU's system calls differ greatly from those of
Linux, which is why XNU system call emulation is at the heart of Darling.

## XNU system calls

Unlike other kernels, XNU has three distinct system call tables:

 1. **BSD system calls**, invoked via sysenter/syscall. These calls frequently
    have a direct counterpart on Linux, which makes them easiest to implement.
 2. **Mach system calls**, which use negative system call numbers. These are
    non-existent on Linux and are mostly implemented in the darling-mach Linux
    kernel module. These calls are built around [Mach
    ports](../macos-specifics/mach-ports.md).
 3. **Machine dependent system calls**, invoked via `int 0x82`. There is only a
    few of them and they are mainly related to [thread-local
    storage](../threading/thread-local-storage.md).

## Introduction

Darling emulates system calls by providing a modified
`/usr/lib/system/libsystem_kernel.dylib` library, the source code of which is
located in `src/kernel`. Even though some parts of the emulation are located in
Darling's kernel module (located in `src/external/lkm`), Darling's emulation is
user space based.

This is why `libsystem_kernel.dylib` (and also `dyld`, which contains a static
build of this library) can never be copied from macOS to Darling.

Emulating XNU system calls directly in Linux would have a few benefits, but
isn't really workable. Unlike BSD kernels, Linux has no support for foreign
system call emulation and having such an extensive and intrusive patchset merged
into Linux would be too difficult. Requiring Darling's users to patch their
kernels is out of question as well.

### Disadvantages of this approach

* Inability to run a full copy of macOS under Darling (notwithstanding the
  legality of such endeavor), at least the files mentioned above need to be
  different.

* Inability to run software making direct system calls. This includes Go
  applications and some UPX packed executables. Note that [Apple provides no
  support](https://developer.apple.com/library/content/qa/qa1118/_index.html)
  for making direct system calls (which is effectively very similar to
  distributing statically linked executables described in the article) and
  frequently changes the system call table, hence such software is bound to
  break over time.

### Advantages of this approach

* Significantly easier development.
* No need to worry about having patches merged into Linux.
* It is easier for users to have newest code available, it is not needed to run
  the latest kernel to have the most up to date system call emulation.

## Implementation

TODO
