# Thread implementation

## Rationale

Darling uses Apple's original libpthread library. MacOS applications running
under Darling therefore use the same exact threading library as on macOS.

Apple's libpthread manages threads through a collection of `bsdthread*` system
calls, which are [implemented by
Darling](https://github.com/darlinghq/darling/tree/master/src/kernel/emulation/linux/bsdthread).
This way Apple's libpthread could operate absolutely independently on Linux.

However, there is a huge catch. Darling could set up threads on its own and
everything would be working fine, but only unless no calls to native Linux
libraries (e.g. PulseAudio) would be made. The problem would become obvious as
soon as given Linux library would make any thread-related operations on its
own - they would crash immediately. This includes many `pthread_*` calls, but
thread-local storage access as well.

## Wrapping native libpthread

In Darling, `bsdthread*` system calls call back into the
[loader](../basics/loader.md), which
[uses](https://github.com/darlinghq/darling/blob/master/src/startup/threads.c)
native libpthread to start a thread. Once native libpthread sets up the thread,
the control is handed over to Apple's libpthread.

This carries its own share of complications. Apple's libpthread needs control
over the stack, hence we cannot use the stack provided and managed by native
libpthread. Furthermore, someone needs to call `pthread_join` on the native
Linux pthread object in order to free all associated resources.

This is why there is a special "reaper" thread fired up once the macOS
application starts using threads. Note that starting additional threads not
directly initiated by the application - such as the reaper thread - is of no
concern on Darling. The use of Grand Central Dispatch (libdispatch) involves
possible creation of many unsolicited threads, so applications are not expected
to treat additional threads as a sign of "cracking", like some Windows apps may
do.

