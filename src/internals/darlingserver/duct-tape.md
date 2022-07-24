# Duct-Taped Code

As described in the [section intro](./README.md), darlingserver uses XNU code
to implement much of the functionality needed for server calls. However, because
this code is kernel code and uses code from several other kernel subsystems that
we can't include in darlingserver, we have to implement some support code that
essentially duct-tapes the XNU code into something we can use.

## Hooks

Because duct-taped code uses XNU kernel headers, including any Linux headers
would quickly cause header conflicts and complications. Therefore, whenever
our duct-taping code needs to use some Linux functionality, we do one of two
things: either manually add declaractions/prototypes (e.g. for simple things
like `mmap`) or, more commonly, add hooks for duct-taped code to call into C++
code. Additionally, duct-taped code is C code, so if it needs to use some
functionality that the C++ code implements, it *must* use hooks.

Currently, there are quite a lot of hooks for various things. The most common
types of hooks are hooks for process- and thread-related functionality
implemented in C++ code. There are also hooks for microthreading features like
suspension, resumption, termination, and kernel microthread creation.
Additionally, there are hooks for various other miscellaneous features like
timer creation and logging.

## Locks

Because duct-taped code runs within microthreads, it can't use normal locks.
Instead, we implement microthread-aware locks that suspend the microthread
when they need to sleep instead of blocking the worker thread running the
microthread.

These duct-taped locks use a normal lock to protect their internal state.
However, this normal lock is only held for short periods of time (only to manage
the duct-taped lock state), so it doesn't significantly block the worker thread.

In order to lock a duct-taped lock, the internal state lock is first acquired.
Once we have that lock, we check the duct-taped lock state to see if the lock is
free. If it is, we mark it as locked by our current microthread, unlock the
state lock, and then return. If it's not free, then we add ourselves to the
lock wait queue so that whoever currently owns the lock can wake us up when
they're done, and then we suspend our microthread. Note that the lock wait queue
is just a simple queue of XNU thread structures; we cannot use XNU's `waitq`
for this because `waitq`s themselves need to use locks.

When unlocking a duct-taped lock, we first acquire the internal state lock.
Then we mark the lock as unlocked and check if anyone is waiting for the lock.
If someone is waiting for the lock, we remove them from the queue and wake
them up. Finally, we unlock the state lock and return.

XNU uses different types of locks for different purposes (e.g. spin locks,
mutexes, `simple_lock`s, `ulock`s, and more). However, for our purposes,
all locks are mutexes implemented as described above.

## Timers

Timers are essential for duct-taped XNU code. Fortunately for us, XNU implements
them by overlaying an architecture-independent layer onto a very thin
architecture-dependent layer; all we have to implement is that
architecture-dependent layer.

For this, all we have to do set up a timerfd in the event loop and provide a
hook for duct-tape code to arm/disarm the timer. When the timer fires, we notify
the duct-tape code that it fired, who then notifies the XNU
architecture-independent timer layer.
