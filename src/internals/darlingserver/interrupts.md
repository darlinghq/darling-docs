# Interrupts/Signals

darlingserver needs to be involved in signal processing for two main reasons.
The first is that there is some Mach functionality that relies on having
access to signal information such as what signal occurred and the register state
when it occured so that we can pass this on to debuggers when they ask for it.
The second reason is that signals can occur while a thread is waiting for a
server call to complete. When this occurs, there is a possible race between
the server replying to thread's interrupted call and the thread making a new
server call as part of signal processing/handling.

## Informing the Server about Signals

When a thread is interrupted/signaled, it informs the server using the
`interrupt_enter` server call (more on that in
[Handling Interrupted Calls](#handling-interrupted-calls)). This method only
informs the server that the thread has been interrupted and allows the client
to wait for the server to sort things out on its end (in case it was in the
middle of processing the client's interrupted call). A separate call,
`sigprocess`, is required to inform the server about the signal info and
register state.

On the server side, this call reads the signal information from the client
and hands it over to some duct-taped XNU signal processing code. This code
notifies anyone that's interested (e.g. debuggers) and allows them to manipulate
the thread state if necessary. Once that's done, the server writes out the new
thread state to the client and sends the reply to allow the client to continue.
The client then processes the signal as appropriate (e.g. calling signal
handlers installed by program code), and once it's done, it informs the server
using `interrupt_exit`.

## Handling Interrupted Calls

One issue with handling signals is that signal handlers need to make server
calls of their own, but signals can occur at any time, even in the middle of
another server call. If this is not handled correctly, a race condition is
almost sure to occur, with the server sending a reply for the interrupted call
at the same time that the client makes a new call in the signal handler.

That's why the `interrupt_enter` call also synchronizes the call state with the
server. What this means is that it informs the server that the client received
a signal and waits for the server to give it the green light to continue.

On the server side, if the server was processing a call for the client, the call
is aborted and any reply it generates is saved to be sent once the client
finishes processing the signal (i.e. when it makes the `interrupt_exit` call).

However, there's still another possible source for a race condition: what if
the server had just finished processing the call and had already queued the
reply (or had just finished sending it) when the client received a signal and
made the `interrupt_enter` call? In this case, the client would immediately
receive a reply, but for the wrong call (the one that was just interrupted).
That's why another job of `interrupt_enter` on the client side is to handle
unexpected replies and push them back to the server.

It allocates a buffer (on the stack) large enough to accomodate all possible
replies and file descriptors. When it receives a reply it did not expect (i.e.
a reply that is not for the `interrupt_enter` call), it sends it back to the
server using a special `push_reply` message, which the server uses to save the
message so it can send it once it receives an `interrupt_exit` call.
