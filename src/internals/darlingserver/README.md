# darlingserver

darlingserver is Darling's userspace kernel server, much like wineserver for Wine.
It implements some services that the XNU kernel would normally provide on macOS.

Because of the types of services it provides and the way it implements some of those services,
darlingserver is very low-level and is a fairly complex beast. This can be a major hurdle for new
developers and contributors. Therefore, the goal of this documentation is to explain as much of darlingserver's
architecture and implementation as possible to make it easier to work on.

## Basic Design and Architecture Decisions

### Use Parts of XNU When Possible

The first and most important thing to understand about darlingserver's architecture is that it includes part of the XNU kernel as part of its sources.
The reasoning behind this decision can be summed up as "why reinvent the wheel when you can just use it instead?"

XNU is open-source and implements many of the services darlingserver needs to provide, so we can just use it to implement them.
In fact, we've actually already tried reinventing the wheel in the past with an older version of the LKM by implementing those services
ourselves. It turned out to be much easier to just duct-tape the necessary XNU code, which was the approach used in the now-defunct LKM.

### Avoid Wasting Resources

This sounds like a rather obvious decision, but with something as complex as darlingserver,
this is certainly a conscious design choice that must be kept in mind for all other design and implementation decisions.
While darlingserver is certainly not as optimized as it could be, it strives to avoid wasting resources like file descriptors, CPU time, threads, and memory, among other things.

For example, rather than create a real Linux thread for each thread the server manages, we use userspace threads called microthreads (a.k.a. fibers) and perform cooperative
scheduling between them ourselves. Using real Linux threads would greatly simplify thread management within darlingserver, but Linux threads are much heavier resource-wise
and creating one for each thread the server manages would waste too many resources unnecessarily.

## Actual Design and Implementation

With the overarching design decisions in mind, it should be easier to understand why certain things
are implemented the way they are. Feel free to read up on these topics in whatever order you like:

  * [RPC](rpc.md) (including S2C calls)
  * [Threads, processes, and microthreads](threads-and-friends.md)
  * [Duct-taped code](duct-tape.md)
  * [kqueue channels](kqchannels.md)
  * [Interrupt/signal handling](interrupts.md)
  * [Event loop](event-loop.md)
