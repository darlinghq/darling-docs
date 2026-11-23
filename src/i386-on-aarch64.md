# Running i386 programs on aarch64 (experimental)

An aarch64 Linux host cannot run the 32-bit x86 (i386) slice of a Mach-O file by itself. `mldr` only picks arm64 or arm slices there, so a universal file that has an i386 slice but nothing the host can run fails. Without the support described below, the message is `No supported architecture found in fat binary.` Thin (single-architecture) i386 files are not covered.

This page describes how to run such a program with an x86 emulator that runs 32-bit Linux programs, with [FEX](https://github.com/FEX-Emu/FEX) as the tested example. It is experimental. Only one application has been tried, and audio, joysticks, hardware GPU rendering and performance have not been tested.

## How it works

When a universal file has an i386 slice but no slice the host can run, `mldr` looks at the `DARLING_I386_EMULATOR` environment variable. If it holds the path of an emulator (`mldr` uses `execv`, so a bare command name is not searched for), `mldr` runs `mldr32` under it:

```
$DARLING_I386_EMULATOR <path of mldr32> <the arguments mldr was given, without argv[0]>
```

The program's path is passed in the `__mldr_exec_path` environment variable, because an emulator sets the program's `argv[0]` itself. The emulator must therefore pass its environment through unchanged.

`mldr32` must sit next to `mldr` (its path is the path of `mldr` with `32` added) and must be an i386 ELF binary built from the same source as `mldr`, because an older `mldr32` does not know `__mldr_exec_path`. The emulator must run i386 Linux programs, so it needs an i386 root file system with the libraries `mldr32` and the program's host-side dependencies (X11, Mesa and so on) load.

Without the variable, `mldr` prints a message that names it and exits.

## Host requirements found so far

These are properties of the host and the emulator, not of Darling:

- **4K pages.** FEX's allocator refuses a kernel with 16K pages, such as the one used on Apple Silicon hosts. Running FEX inside a virtual machine with 4K pages, for example with `muvm`, avoids this.
- **`vm.mmap_min_addr`.** The application tried here has its `__TEXT` segment at `0x1000`. With the value 32768 that the virtual machine had, mapping it fails with `Operation not permitted` (see also [Common issues](common-issues.md)). Lower it to 4096 inside the virtual machine only.
- **Top of the 32-bit address space.** `mldr32` maps the commpage at `0xffff0000`. With the FEX build first used, that range was already taken (`Cannot mmap commpage: File exists`). Getting past it needed a local change to how FEX reserves the top of a 32-bit guest's address space. That change is not in upstream FEX, so this setup currently cannot be reproduced from upstream FEX alone.
- **i386 libraries.** Provide i386 builds of glibc, libstdc++, X11 and (for OpenGL programs) Mesa, plus any other library the program's frameworks load, in the emulator's root file system. A missing one shows up as, for example, `Cannot load libX11.so.6 (ELF): wrong ELF class: ELFCLASS64`. The program tried here also needed libpng, libdbus and `libjpeg.so.8`; the last one is not in Debian and came from Ubuntu's i386 packages.
- **Shared memory with X.** If the emulated program runs in a virtual machine and the X server runs on the host, MIT-SHM does not work across that boundary and windows stay black. Use an X server with the MIT-SHM extension disabled, for example `Xvfb -extension MIT-SHM` or a rootful `Xwayland` started the same way.

## Known problems

- Programs can block forever in `sendto()` to the prefix's syslog socket (`var/run/syslog`). This also happens without an emulator: native Darling daemons were seen waiting on the same socket in the aarch64 virtual machine used here. In one prefix, seven daemons each logged two `libdispatch` messages at startup: `kevent monitor ... Function not implemented` (the memorystatus kevent) and a `25E246 - 5830 - 0x16` BUG message. That left 14 datagrams unread in the socket, and the limit for a unix datagram queue (`net.unix.max_dgram_qlen`, 10 by default) was exceeded. `_simple_asl_send` in `libplatform` sends with a blocking `sendto()` and no timeout, so later senders blocked. `syslogd` read the whole backlog after part of it was drained from outside and a new datagram arrived. It is not known why it did not read it by itself. Setting `ASL_DISABLE=1` in the program's environment only hides the problem.

## Status

Tried: Sky Fighters 1945, a universal file with ppc and i386 slices. It was started with `DARLING_I386_EMULATOR` set to a FEX build, and with an `mldr32` built from the same source as `mldr`. It reached its trial and instructions dialogs, the player sign-in and a 3D flight scene rendered in software. In that run the dialogs were driven with synthetic X events.

In an earlier setup, which used a wrapper program in place of `mldr`, the same game was also used with a real keyboard and mouse through an Xwayland window, and it produced sound. Neither the sound nor the real input path was re-checked through `DARLING_I386_EMULATOR`. Joysticks, hardware GPU rendering and performance have not been tested.
