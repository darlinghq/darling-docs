# Distributed Objects

Here's how the Distributed Objects are structured internally:

* **`NSPort`** is a type that abstracts away a *port* -- something that can
  receive and send messages (`NSPortMessage`). The few commonly used concrete
  port classes are `NSMachPort` (Mach port), `NSSocketPort` (network socket,
  possibly talking to another host), and `NSMessagePort` (another wrapper over
  Mach ports). With some luck, it's possible to use a custom port class, too.
  
  `NSPort` is really supposed to be a Mach port (and that's what `[NSPort port]`
  creates), while other port types have kind of been retrofitted on top of the
  existing Mach port semantics. `NSPort` itself conforms to `NSCoding`, so you
  can "send" a port over another port (it does not support coders other than
  `NSPortCoder`).
  
  Some `NSPort` subclasses may not fully work unless you're using them for
  Distributed Objects (with an actual `NSConnection *`).

* **`NSPortMessage`** roughly describes a Mach message. It has "send" and
  "receive" ports, a msgid, and an array of *components*. Individual components
  can be either data (`NSData`) or a port (`NSPort`), corresponding to
  `MACH_MSG_OOL_DESCRIPTOR` and `MACH_MSG_PORT_DESCRIPTOR`. Passing a port will
  only work with ports of the same type as the port you're sending this message
  through.

* **`NSPortNameServer`** abstracts away a name server that you can use to map
  (string) names to port. You can register your port for a name and lookup other
  ports by name. `NSMachBootstrapServer` implements this interface on top of the
  Mach bootstrap server (**launchd** on Darwin).

* **`NSPortCoder`** is an `NSCoder` that essentially serializes and deserializes
  data (and ports) to and from port messages, using the same `NSCoding`
  infrastructure a lot of types already implement. Unlike other coders (read:
  archivers), it supports encoding and decoding ports, though this is mostly
  useless as DO itself make very little use of this.
  
  `NSPortCoder` itself is an abstract class. In older versions of OS X, it had a
  concrete subclass, `NSConcretePortCoder`, which only supported non-keyed (also
  called unkeyed) coding. In newer versions, `NSConcretePortCoder` itself is now
  an abstract class, and it has two subclasses, `NSKeyedPortCoder` and
  `NSUnkeyedPortCoder`.
  
  `NSPortCoder` subclasses send the `replacementObjectForPortCoder:` message to
  the objects they encode. The default implementation of that method replaces
  the object with a `NSDistantObject` (a Distributed Objects proxy), no matter
  whether the type conforms to `NSCoding` or not. Some DO-aware classes that
  wish to be encoded differently (for example, `NSString`) override that method,
  and usually do something like this:
  
  ```objc
  - (id) replacementObjectForPortCoder: (NSPortCoder *) portCoder {
      if ([portCoder isByref]) {
          return [super replacementObjectForPortCoder: portCoder];
      }
      return self;
  }
  ```

* **`NSDistantObject`** is a proxy (`NSProxy`) that stands in for a remote
  object and forwards any messages over to the remote object. The same
  `NSDistantObject` type is returned by the default `NSObject` implementation of
  `replacementObjectForPortCoder:`, and this instance is what gets serialized
  and sent over the connection (and deserialized to a `NSDistantObject` on the
  other end).

* **`NSConnection`** represents a connection (described by a pair of ports).
  `NSConnection` stores local and remote object maps, to intern created proxies
  (`NSDistantObject`s) when receiving or sending objects. `NSConnection`
  actually implements forwarding method invocations, and also serves as a port's
  delegate, handling received messages.

You would think that `NSPort`, `NSPortMessage`, `NSPortNameServer`, and
`NSPortCoder` do not "know" they're being used for DO/RPC, and are generic
enough to be used for regular communication directly. This is *almost* true, but
DO specifics pop up in unexpected places.

# Resources

*  https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/DistrObjects/DistrObjects.html
*  GNUstep implementation
