# Threads, Processes, and Microthreads

darlingserver needs to keep track of the threads and processes it manages because both darlingserver code and XNU code require this information.
Additionally, we use microthreads to execute calls and run XNU code in the context of a particular thread.

## Lifetime Management

When a thread or process is created, the first thing it does is to create a new client socket and check-in with the server to register
itself with the server. On the server side, a new thread/process is created and some initialization is performed to set up an XNU context
for the thread/process.

Before a thread exits, it informs the server that it's checking-out and the server performs some cleanup of the info it had on the thread.
With a process exiting, however, the server is instead notified by the system via a `procfd` that the server opens for the process when it
registers a new process. When the server is informed that a process has died, it cleans up any server-side threads that the process might
still have registered; this can happen, for example, when the program is killed by a signal and doesn't have a chance to inform the server
that its threads are dying.

## Detecting Forks and Execs

Detecting forks is quite simple: when a client forks, it has to replace its
existing socket with a new one so it can have its own separate connection
to the server instead of inheriting its parents connection; in doing so, it
has to check-in with the server again, so darlingserver can find out that way.

Detecting `exec`s is a little more complicated. When a client is going to
perform an `exec`, it opens a close-on-exec pipe and gives it to the
server so that it can wait for an event on this pipe. If the client successfully
performs the `exec`, the pipe is automatically closed by Linux and the server
will receive a hang-up event on the pipe. When it sees that the pipe is empty,
it knows that the `exec` succeeded and the process has been replaced. However,
if the `exec` fails, the pipe is not automatically closed by Linux. Instead,
the client writes a single byte into the pipe and then closes it. When the
server receives the hang-up event and sees the pipe is not empty, it knows the
`exec` failed and the process has not been replaced.

## Reading From and Writing To Process Memory

One thing that is often necessary when implementing RPC calls is the ability to read and write directly to/from process memory.
Often, this is used to access memory at a pointer given as part of a call argument. This can be achieved using the `readMemory` and `writeMemory`
functions available in `Process` instances. These methods make use of Linux's `process_vm_readv` and `process_vm_writev` to access the target
process' memory.

## Identifiers

All Darling threads and processes have two identifiers: one that darlingserver sees (the one in its PID namespace) and one that other Darling processes
see (the one within the Darling PID namespace). In darlingserver, the first type of ID is just called an ID (identifier); it is sometimes referred
to as the global or Linux ID, as well. The second type of ID is called an NSID (namespaced identifier).

[Kernel microthreads](#kernel-microthreads) have NSIDs but they do *not* have IDs because they have no corresponding Linux thread/process.
Additionally, kernel microthread NSIDs are always very high numbers to more easily identify them for debugging purposes. This distinction is not
enough to identify a microthread as being a kernel microthread, however; rather, you can tell if a Thread is a kernel microthread if its ID is invalid
(e.g. `-1`) but its NSID valid.

## Thread and Process Registries

darlingserver keeps track of all active processes and threads in two registries (`processRegistry` and `threadRegistry`). Each registry
can be accessed via either an ID or an NSID.

The process registry does include the kernel process, as well. This is a pseudo-process that does not have an actual Darling process associated with it;
it exists mainly for duct-taped XNU code to use and to own all the kernel microthreads. The thread registry also includes all kernel microthreads.

## Microthreading

A major component of darlingserver is microthreading. This consists of executing multiple threads using cooperative scheduling within one or more
Linux worker threads. As described in the [section intro](./README.md), this approach was chosen because it conserves resources. Currently, due to bugs
in the multithreaded implementation, only a single worker thread is used.

Microthreads use `setcontext`, `makecontext`, and `getcontext` to perform cooperative multitasking. Whenever a microthread needs to wait for something,
it suspends itself and returns to the worker thread runner; the worker thread runner function is considered the "top" of the thread (and you will see
mentions of this in the microthreading code). Microthreads can also optionally suspend with a custom continuation point. By default, when a microthread
suspends itself, it will continue executing where it left off once it resumes. However, with a custom continuation point, the old stack is discarded
and the microthread will start executing from the continuation point once it resumes. This functionality is used extensively by duct-taped XNU code.

All the information for each microthread is contained within a Thread instance. When running in a microthread, the current Thread and Process instances
can be accessed via the `currentThread` and `currentProcess` static methods on each respective class.

To further conserve resources, microthreads use a stack pool to limit the number of simultaneously allocated stacks. With this stack pool,
we avoid wasting memory on stacks that are currently unused (only microthreads currently executing or suspended normally need stacks) and we
also avoid the (slight) additional delay of allocating a brand new stack every time we need one.

### Details on the Worker Thread Runner

The worker thread runner function itself is actually very simply: it just invokes the `doWork` method on the Thread instance it was given.
This `doWork` method is where the core of the microthreading logic is.

The first thing this method does is check whether the microthread is even allowed to run: microthreads can be be deferred, already running,
terminating, or even dead; if the microthread is in any of these states, `doWork` simply returns.

The next thing this method does is set up some context for the microthread to run: it sets the `_running` flag for the Thread instance,
sets the Thread instance as the current Thread (a thread-local variable is used to keep track of this for each worker thread), and notifies
the duct-tape code that this Thread is going to start running.

Then, this method uses `getcontext` to initialize the `ucontext` for the microthread to switch back to when it wants to suspend; a thread-local variable
is used for each worker to store this context. However, the catch with `getcontext`/`setcontext` compared to `setjmp`/`longjmp` is that they give no
indication of whether the context is being executed for the first time or whether it has been re-executed (e.g. using `getcontext`). Therefore, an
additional thread-local flag for each worker is used to keep track of this. When the method is first invoked, it sets this flag to `false`; whenever
a microthread jumps back to the worker thread runner, it sets this flag to `true` so the worker thread runner can change its behavior accordingly.

That's why the next thing this method does is check whether the flag is set; if it is, then the microthread has already run and has returned back to the
worker thread runner. In this case, the method performs some clean up: it unsets the `_running` flag on the Thread instance, unsets it as the current
thread, and notifies the duct-tape code that the microthread has finished running. The method also checks whether the Thread has been
marked for termination. If the microthread has instructed us to unlock a certain lock once it's fully suspended, this method also takes care of that now;
this approach is used to ensure the microthread doesn't miss wake-ups from anything protected by that lock.

In the case where the "returning to thread top" flag is unset, this method examines the Thread to determine where it should start executing it.
The first thing it checks is whether there's a pending `interrupt_enter` call. If so, it saves the current Thread state and creates a new one
to begin processing the interrupt (see [Interrupts/Signals](./interrupts.md) for more info). Otherwise, it checks if the thread has a pending call.
If it does, the old stack is discarded (which includes any suspended state) and a new stack is created to being processing the incoming call.
Finally, if the thread has no pending call and is suspended, this method resumes the thread from the appropriate point of execution: if it had a pending
continuation callback, it resumes from the continuation callback; otherwise, it resumes from the point where it was suspended. In all cases, this
branch ("returning to thread top" being unset) does not continue executing normally; it *always* switches a new context to begin executing the thread.

## Kernel Microthreads

Sometimes, it is necessary to run code within a microthread that does not have a managed thread associated with it; this is most necessary for
duct-taped XNU code. That's why darlingserver also supports kernel microthreads. These are Thread instances that have the necessary state for
running code (usually duct-taped code) but do not have a managed thread. This implies that some information on these threads is unavailable.
For example, these threads cannot perform S2C calls, process interrupts, or process calls of their own. However, they can still be schedule to run and
can suspend and resume like normal threads.

In most cases, C++ code doesn't need to use kernel microthreads directly. Duct-taped XNU code is usually the one that wants to create kernel threads
for things like deferred clean up, `thread_call`s, and `timer_call`s. C++ code usually uses `Thread::kernelAsync` and, less commonly,
`Thread::kernelSync`. These static methods allow you to schedule a lambda to run within a kernel microthread at some point in the future. As the names
imply, one schedules it asynchronously&mdash;schedule it to run and then return normally&mdash;and the other schedules it synchronously&mdash;schedule
it to run and wait for it to finish executing before returning. Note that the synchronous method should **never** be called from another microthread (at
least with the current implementation). This is because it will block the worker thread and prevent it from switching to any other microthread. For the
same reason, it should never be called from the main (event loop) thread, as this thread needs to be constantly handling new events.
