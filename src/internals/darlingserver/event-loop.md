# Event Loop

The main thread is in charge of the event loop for darlingserver. This event
loop handles things like sending and receiving messages, detecting process
deaths, and handling timers.

## Sending and Receiving Messages

All messages are sent and received asynchronously. The MessageQueue class wraps
up this functionality neatly and makes it thread-safe to push and pop messages
to/from the queue. Messages are sent and received in batches using `sendmmsg`
and `recvmmsg`.

## Timers

The event loop includes a timerfd for duct-taped code to arm and disarm as
necessary. See the [duct-tape timers](./duct-tape.md#timers) section for more
info.

## Process Deaths

When a process is registered with the server, darlingserver opens a `procfd` for
the process and adds it to its epoll context. Linux marks the `procfd` as
readable once the target process dies.

## Monitors

darlingserver also includes support for adding generic descriptors to the event
loop via the Monitor class. Monitors allow you to add a descriptor to the event
loop and get notified via a lambda that runs on the main (a.k.a. event loop)
thread when the event occurs.
