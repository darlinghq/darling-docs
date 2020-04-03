# Distributed Objects

Here's how the Distributed Objects is structured internally:


*  **''NSPort''** is a type that abstracts away a *port* -- something that can receive and send messages (''NSPortMessage''). The few commonly used concrete port classes are ''NSMachPort'' (Mach port), ''NSSocketPort'' (network socket, possibly talking to another host), and ''NSMessagePort'' (Unix domain socket). With some luck, it's possible to use a custom port class, too. ''NSPort'' is really supposed to be a Mach port (and that's what ''[NSPort port]'' creates), while other port types have kind of been retrofitted on top of the existing Mach port semantics. ''NSPort'' itself conforms to ''NSCoding'', so you can "send" a port over another port (it does not support coders other than ''NSPortCoder'').

*  **''NSPortMessage''** roughly describes a Mach message. It has "send" and "receive" ports, a msgid, and an array of *components*. Individual components can be either data (''NSData'') or a port (''NSPort''), corresponding to ''MACH_MSG_OOL_DESCRIPTOR'' and ''MACH_MSG_PORT_DESCRIPTOR''. Passing a port will only work with ports of the same type as the port you're sending this message through.

*  **''NSPortNameServer''** abstracts away a name server that you can use to map (string) names to port. You can register your port for a name and lookup other ports by name. ''NSMachBootstrapServer'' implements this interface on top of the Mach bootstrap server (**launchd** on Darwin).

*  **''NSPortCoder''** is an ''NSCoder'' that essentially serializes and deserializes data (and ports) to and from port messages, using the same ''NSCoding'' infrastructure a lot of types already implement. Unlike other coders (read: archivers), it supports encoding and decoding ports, though this is mostly useless as DO itself doesn't make use of this. ''NSPortCoder'' doesn't support keyed coding. It also sends ''replacementObjectForPortCoder:'' (instead of the usual ''replacementObjectForPortCoder:'') to objects being encoded.

*  **''NSDistantObject''** is a proxy (''NSProxy'') that stands in for a remote object and forwards any messages over to the remote object. The same ''NSDistantObject'' type is returned by the default ''NSObject'' implementation of ''replacementObjectForPortCoder:'', and this instance is what gets serialized and sent over the connection (and deserialized to a ''NSDistantObject'' on the other end). ''NSDistantObject'' really wants to be serialized or deserialized as a part of an ''NSConnection''.

*  **''NSConnection''** represents a connection (described by a pair of ports). ''NSConnection'' stores local and remote object maps, to intern created proxies (''NSDistantObject''s) when receiving or sending objects. ''NSConnection'' actually implements forwarding method invocations, and also serves as a port's delegate, handling received messages.

Notably, ''NSPort'', ''NSPortMessage'', ''NSPortNameServer'', and ''NSPortCoder'' do not "know" they're being used for DO/RPC, and can be used for regular communication directly, with the caveat that ''NSPortCoder'' will attempt to serialize ''NSDistantObject''s instead of most objects, and that will fail unless there is an actual ''NSConnection''.

# Resources


*  https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/DistrObjects/DistrObjects.html

*  GNUstep implementation
