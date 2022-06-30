# `/dev/mach` character device and LKM traps

The LKM needs to keep track of the processes it manages, for various reasons (for example, some parts of XNU that we use need this information).
Additionally, the LKM needs to provide traps for userspace to perform certain operations that require kernel-space intervention (such as Mach IPC).

The simplest way to satisfy both these requirements is for each Darling process to have a special character device managed by our kernel module.
When a process is created, the LKM opens this character device (`/dev/mach`) and gives it to the process. After this, the LKM can track when the process
forks, execs, or exits. Additionally, this character device can be used by the LKM to handle userspace requests (i.e. traps).

## Detecting forks, execs, and exits

As mentioned before, the LKM needs to keep track of forks, execs, and exits.

Detecting an exit is pretty simple: Linux tells us when our character device is being closed, and Linux closes all open files for process when it exits.
So as long as we detect it's not an exec (details on that later), we know it's an exit.

Forks are a little more complicated. The child process inherits the parent's descriptor (just like every other descriptor).
When the child performs its first LKM trap (which is guaranteed to happen in libsystem_kernel as part of post-fork initialization),
the LKM checks the process information it has associated with the descriptor, sees that it belongs to a different process, and swaps in a new descriptor for the child.

Execs are also tricky, as they have to do with the way we support Mach-O loading.
An important detail is that when the LKM creates a new "mach" chardev descriptor for a process, it sets O_CLOEXEC on the descriptor.
When the process performs an exec for new Mach-O, Linux performs some minimal setup and then hands over control to our binfmt loader.
Our loader loads the new binary, sets up a brand new XNU task for it, and then calls Linux's `begin_new_exec` (or `flush_old_exec` on older kernels).
This Linux kernel function is responsible for completely tearing down the old executable and registering the new one.
This is also where Linux closes all O_CLOEXEC descriptors for the old image. Thus, when our chardev's close handler is called here,
the XNU task associated with the current process is a brand new one. The chardev close handler can detect this and thus it knows that we must be performing an exec.

Note an additional implication from the exec detection method we employ: when a Mach-O process execs itself into an ELF binary, the LKM sees it as the Mach-O binary exiting normally.

## Invoking traps from userspace

An important role of the LKM is to fulfill userspace requests (a.k.a. traps) for things like Mach IPC, debugger support, Mach semaphore, kqueue, and more.
This is one of the main jobs of the character device. Using ioctls on the chardev, userspace can call into the LKM. Our LKM provides [an ioctl handler](https://github.com/darlinghq/darling-newlkm/blob/master/darling/traps.c#L668)
for the chardev which looks up the request trap and invokes it.

## Passing memory around in traps

Many (but not all) traps require additional arguments. Due to these arguments being in a different memory space (i.e. they come from userspace, but we're operating in kernel-space),
traps must copy-in pointer arguments using the [`copyargs` macro](https://github.com/darlinghq/darling-newlkm/blob/master/darling/traps.c#L750) (or Linux's `copy_from_user`).
Likewise, to pass memory back to userspace, traps must use Linux's `copy_to_user`.
