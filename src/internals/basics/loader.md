# Loader

Whereas Linux uses the
[ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) as the format for applications, dynamic libraries and so on, macOS uses [Mach-O](https://en.wikipedia.org/wiki/Mach-O).

[Darling's Linux kernel module](https://github.com/darlinghq/darling-newlkm) contains a static [loader](https://github.com/darlinghq/darling-newlkm/blob/master/darling/binfmt.c) of Mach-O binaries, whose primary task is to load the application's binary and then load `dyld` and hand over control to it.

An additional trick is then needed to [load ELF libraries](../calling-host-system-apis.md) into such processes.

## dyld

Dyld is Apple's dynamic linker. It examines what libraries are needed by the
macOS application, loads them and performs other necessary tasks. After it is
done, it jumps to application's entry point.

Dyld is special in the sense that as the only "executable" on macOS, it does
not (cannot) link to any libraries. As a consequence of this, it has to be
statically linked to a subset of libSystem (counterpart of glibc on Linux).
