# Loader

Multiple loaders are involved in loading macOS applications on Linux. This is due to the fact that macOS applications use [Mach-O](https///en.wikipedia.org/wiki/Mach-O) as the file format of applications, dynamic libraries and so on, whereas Linux uses the [ELF](https///en.wikipedia.org/wiki/Executable_and_Linkable_Format) format.

## mldr

[mldr](https///github.com/darlinghq/darling/blob/master/src/startup/mldr.c) is the first loader. It's an ordinary Linux executable written in C. Its main purpose is to map the Mach-O executable to be started into the memory, find out what dynamic linker is required to fully load it (''/usr/lib/dyld'' is used on macOS exclusively), load the respective dynamic linker and hand control over to it.

mldr's secondary responsibility is to provide a gateway to the Linux and ELF world. This involves aiding in [thread management](documentation/thread_implementation) or loading required native libraries.

### 32 in 64

As an experimental technology, ''mldr'' is now executing 32-bit macOS applications in 64-bit Linux processes. This is not such a daunting task as it may seem (some general ideas are laid out in [this blog article](http://blog.dolezel.info/2017/02/running-32-bit-code-in-64-bit-linux.html)).

This approach has a few advantages. It eliminates the need for having all of Darling's dependencies also in a 32-bit version. This becomes especially pronounced when building Darling on Debian-based systems:


*  Darling builds 64-bit and 32-bit code in one go (so that it can produce fat Mach-O binaries).

*  On Debian, you need separate -dev packages for 64-bit and 32-bit build.

*  These packages cannot be installed at the same time (they conflict).

There are other benefits as well. For instance, it enables us to map the [commpage](documentation/commpage) at the address where it exists on macOS in 32-bit processes, which would not be possible in 32-bit Linux processes, where the commpage address belongs in the kernel-only memory range (upper 1 GB). This problem is non-existent in 64-bit Linux processes.

There are disadvantages as well. Interfacing with native Linux libraries requires some degree of translation to be performed.

## dyld

Dyld is Apple's dynamic linker. It examines what libraries are needed by the macOS application, loads them and performs other necessary tasks. After it is done, it jumps to application's entry point.

Dyld is special in the sense that it as the only "executable" on macOS, it does not (cannot) link to any libraries. As a consequence of this, it has to be statically linked to a subset of libSystem (counterpart of glibc on Linux).


