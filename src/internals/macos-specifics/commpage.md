# Commpage

Commpage is a special memory structure that is always located at the same
address in all processes. On macOS, this mapping is provided by the kernel. In
case of Darling, this functionality is supplemented by the kernel module.

## Purpose

* CPU information (number of cores, CPU capabilities etc.)
* Current precise date and time (this information is not filled in by Darling,
  causing a fall back to a system call).
* PFZ - preemption-free zone. Contains a piece of code which when run prevents
  the process from being preempted. This is used for a lock-free implementation
  of certain primitives (e.g. OSAtomicFifoEnqueue). (Not available under
  Darling.)

It is somewhat related to [vDSO](https://en.wikipedia.org/wiki/VDSO) on Linux,
except that vDSO behaves like a real library, while commpage is just a chunk of
data.

The commpage is not documented anywhere, meaning it's not an API intended to be
used by 3rd party software. It is however used in source code provided on
[opensource.apple.com](http://opensource.apple.com). Darling
[provides](https://github.com/darlinghq/darling-newlkm/blob/master/darling/commpage.c)
a commpage for compatibility reasons.

## Location

The address differs between 32-bit and 64-bit systems.

*  32-bit systems: `0xffff0000`
*  64-bit systems: `0x7fffffe00000`

Note that the 32-bit address is outside of permissible user space area on 32-bit
Linux kernels. This is why Darling runs only under 64-bit kernels, which don't
have this limitation.
