# kqueue Channels

Most kqueue functionality is implemented within
[libkqueue](https://github.com/darlinghq/darling-libkqueue), which runs within
each Darling process. However, there are two filters that cannot be implemented
this way: Mach port filters and process filters. These filters require
information only available within darlingserver.

We can't just use server calls that sleep until the filters become active
because this would completely ignore any other filters in libkqueue. Instead,
we provide server calls to open additional sockets called kqueue channels
(or kqchannels). Libkqueue can add these descriptors to its epoll context
to be woken up when a message is available on the channels. The server writes a
notification message to a kqchannel when an event is available; the client is
woken up and then asks the server for the full event information.

## Mach Port Filters

Mach port filters leverage the existing XNU code for Mach port kqueue filters,communicating back and forth between the duct-taped code and the C++ code.

When a Mach port kqchannel is created, it registers a duct-taped `knote` context
for the Mach port. When the duct-taped Mach port filter code notifies us that
an event is ready, the duct-taped kqchannel code notifies the C++ kqchannel
code, which then notifies the client as described earlier. Likewise, when a
client wants to read the full event details, the C++ code asks the duct-taped
XNU code for event details.

## Process Filters

Process filters hook into the target Process instance in dalringserver to have
it notify us when certain events like forks, execs, and deaths occur.

A special case occurs when a process forks because there is an old, deprecated
feature called `NOTE_TRACK` that allows clients to request for a knote to be 
automatically created for new forks of the target process. Though newer versions
of XNU have completely removed support for this feature, we would like to keep
compatibility with this feature for older software; plus, it's not too hard to
implement.
