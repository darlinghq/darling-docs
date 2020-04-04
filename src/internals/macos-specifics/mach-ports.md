# Mach ports

Mach ports are the IPC primitives under Mach. They are conceptually similar to
Unix pipes, sockets or message queues: using ports, tasks (and the kernel) can
send each other messages.

Drawing the analogy with pipes further,

* A **port** is like a pipe. It is conceptually a message queue maintained by
  the kernel — this is where it differs from a pipe, which is an uninterpreted
  stream of raw bytes.

* Tasks and the kernel itself can enqueue and dequeue messages to/from a port
  via a **port right** to that port. A port right is a handle to a port that
  allows either sending (enqueuing) or receiving (dequeuing) messages, a lot
  like a file descriptor connected to either the read or the write end of a
  pipe. There are the following kinds of port rights:

  * **Receive right**, which allows receiving messages sent to the port. Mach
    ports are MPSC (multiple-producer, single-consumer) queues, which means that
    there may only ever be one receive right for each port in the whole system
    (unlike with pipes, where multiple processes can all hold file descriptors
    to the read end of one pipe).
  * **Send right**, which allows sending messages to the port.
  * **Send-once right**, which allows sending one message to the port and then
    disappears.
  * **Port set right**, which denotes a *port set* rather than a single port.
    Dequeuing a message from a port set dequeues a message from one of the ports
    it contains. Port sets can be used to listen on several ports
    simultaneously, a lot like `select`/`poll`/`epoll`/`kqueue` in Unix.
  * **Dead name**, which is not an actual port right, but merely a placeholder.
    When a port is destroyed, all existing port rights to the port turn into
    dead names.

* A **port right name** is a specific integer value a task uses to refer to a
  port right it holds, a lot like a file descriptor that a process uses to refer
  to an open file. Sending a port right name, the integer, to another task does
  *not* allow it to use the name to access the port right, because the name is
  only meaningful in the context of the *port right namespace* of the original
  task.

The above compares both port rights and port right names to file descriptors,
because Unix doesn't differentiate between the *handle to an open file* and the
*small integer value* aspects of file descriptors. Mach does, but even when
talking about Mach, it's common to say *a port* or *a port right* actually
meaning *a port right name* that denotes the right to the port. In particular,
the `mach_port_t` C type (aka `int`) is actually the type of port right names,
not ports themselves (which are implemented in the kernel and have the real type
of `struct ipc_port`).

## Low-level API

It's possible to use Mach ports directly by using the `mach_msg()` syscall (Mach
trap; actually, `mach_msg()` is a user-space wrapper over the
`mach_msg_overwrite_trap()` trap) and various `mach_*()` functions provided by
the kernel (which are in fact MIG routines, see below).

Here's an example of sending a Mach message between processes:

```c
// File: sender.c

#include <stdio.h>

#include <mach/mach.h>
#include <servers/bootstrap.h>

int main() {

    // Lookup the receiver port using the bootstrap server.
    mach_port_t port;
    kern_return_t kr = bootstrap_look_up(bootstrap_port, "org.darlinghq.example", &port);
    if (kr != KERN_SUCCESS) {
        printf("bootstrap_look_up() failed with code 0x%x\n", kr);
        return 1;
    }
    printf("bootstrap_look_up() returned port right name %d\n", port);


    // Construct our message.
    struct {
        mach_msg_header_t header;
        char some_text[10];
        int some_number;
    } message;

    message.header.msgh_bits = MACH_MSGH_BITS(MACH_MSG_TYPE_COPY_SEND, 0);
    message.header.msgh_remote_port = port;
    message.header.msgh_local_port = MACH_PORT_NULL;

    strncpy(message.some_text, "Hello", sizeof(message.some_text));
    message.some_number = 35;

    // Send the message.
    kr = mach_msg(
        &message.header,  // Same as (mach_msg_header_t *) &message.
        MACH_SEND_MSG,    // Options. We're sending a message.
        sizeof(message),  // Size of the message being sent.
        0,                // Size of the buffer for receiving.
        MACH_PORT_NULL,   // A port to receive a message on, if receiving.
        MACH_MSG_TIMEOUT_NONE,
        MACH_PORT_NULL    // Port for the kernel to send notifications about this message to.
    );
    if (kr != KERN_SUCCESS) {
        printf("mach_msg() failed with code 0x%x\n", kr);
        return 1;
    }
    printf("Sent a message\n");
}
```

```c
// File: receiver.c

#include <stdio.h>

#include <mach/mach.h>
#include <servers/bootstrap.h>

int main() {

    // Create a new port.
    mach_port_t port;
    kern_return_t kr = mach_port_allocate(mach_task_self(), MACH_PORT_RIGHT_RECEIVE, &port);
    if (kr != KERN_SUCCESS) {
        printf("mach_port_allocate() failed with code 0x%x\n", kr);
        return 1;
    }
    printf("mach_port_allocate() created port right name %d\n", port);


    // Give us a send right to this port, in addition to the receive right.
    kr = mach_port_insert_right(mach_task_self(), port, port, MACH_MSG_TYPE_MAKE_SEND);
    if (kr != KERN_SUCCESS) {
        printf("mach_port_insert_right() failed with code 0x%x\n", kr);
        return 1;
    }
    printf("mach_port_insert_right() inserted a send right\n");


    // Send the send right to the bootstrap server, so that it can be looked up by other processes.
    kr = bootstrap_register(bootstrap_port, "org.darlinghq.example", port);
    if (kr != KERN_SUCCESS) {
        printf("bootstrap_register() failed with code 0x%x\n", kr);
        return 1;
    }
    printf("bootstrap_register()'ed our port\n");


    // Wait for a message.
    struct {
        mach_msg_header_t header;
        char some_text[10];
        int some_number;
        mach_msg_trailer_t trailer;
    } message;

    kr = mach_msg(
        &message.header,  // Same as (mach_msg_header_t *) &message.
        MACH_RCV_MSG,     // Options. We're receiving a message.
        0,                // Size of the message being sent, if sending.
        sizeof(message),  // Size of the buffer for receiving.
        port,             // The port to receive a message on.
        MACH_MSG_TIMEOUT_NONE,
        MACH_PORT_NULL    // Port for the kernel to send notifications about this message to.
    );
    if (kr != KERN_SUCCESS) {
        printf("mach_msg() failed with code 0x%x\n", kr);
        return 1;
    }
    printf("Got a message\n");

    message.some_text[9] = 0;
    printf("Text: %s, number: %d\n", message.some_text, message.some_number);
}
```

```	
Darling [/tmp]$ ./receiver
mach_port_allocate() created port right name 2563
mach_port_insert_right() inserted a send right
bootstrap_register()'ed our port

# in another terminal:
Darling [/tmp]$ ./sender
bootstrap_look_up() returned port right name 2563
Sent a message

# back in the first terminal:
Got a message
Text: Hello, number: 35
```

As you can see, in this case the kernel decided to use the same number (2563)
for port right names in both processes. This cannot be relied upon; in general,
the kernel is free to pick any unused names for new ports.

## Ports as capabilities

In addition to custom "inline" data as shown above, it's possible for a message
to contain:

* **Out-of-line data** that will be copied to a new virtual memory page in the
  receiving task. If possible (i.e. if the out-of-line data is page-aligned in
  the sending task), Mach will use copy-on-write techniques to pass the data to
  the receiving task without actually copying it.
* **Port rights** that will be sent to the receiving task. This way, it's
  possible for a task to transfer its port rights to another task. This is how
  `bootstrap_register()` and `bootstrap_look_up()` work in the above example.

The only way for a task to get a port right for a port is to either create that
port, or have some other task (or the kernel) send it the right. In this way,
Mach ports are *capabilities* (as in [capability-based
security](https://en.wikipedia.org/wiki/Capability-based_security)).

Among other things, that means that for Mach programs (including the kernel
itself) which allow other tasks to ask it to perform operations by sending
messages to a port the program listens on, it's idiomatic not to explicitly
check if the sender has any specific "permissions" to perform this task; just
having a send right to the port is considered enough of a permission. For
example, you can manipulate any task (with calls like `vm_write()` and
`thread_create()`) as long as you can get its *task port*; it doesn't matter if
you're *root* or not and so on (and indeed, UIDs and other Unix security
measures don't exist on the Mach level).

## Where to get ports

It would make a lot of sense to make Mach port inheritable across `fork()` and
`exec()` like file descriptors are (and that is the way it work on the Hurd),
but on Darwin, they're not. A task starts with a fresh port right namespace
after either `fork()` or `exec()`, except for a few **special ports** that *are*
inherited:

* **Task port** (aka kernel port), a send right to a port whose receive right is
  held by the kernel. This port allows to manipulate the task it refers to,
  including reading and writing its virtual memory, creating and otherwise
  manipulating its threads, and terminating (killing) the task (see
  [task.defs](https://github.com/darlinghq/darling/blob/master/platform-include/mach/task.defs),
  [mach_vm.defs](https://github.com/darlinghq/darling/blob/master/platform-include/mach/mach_vm.defs)
  and
  [vm_map.defs](https://github.com/darlinghq/darling/blob/master/platform-include/mach/vm_map.defs)).
  Call `mach_task_self()` to get the name for this port for the caller task.
  This port is only inherited across `exec()`; a new task created with `fork()`
  gets a new task port (as a special case, a task also gets a new task port
  after `exec()`ing a suid binary). The Mach `task_create()` function that
  allows creating new tasks returns the task port of the new task to the caller;
  but it's unavailable (always returns `KERN_FAILURE`) on Darwin, so the only
  way to spawn a task and get its port is to perform the ["port swap
  dance"](https://robert.sesek.com/2014/1/changes_to_xnu_mach_ipc.html) while
  doing a `fork()`.

* **Host port**, a send right to another port whose receive right is held by the
  kernel. The host port allows getting information about the kernel and the host
  machine, such as the OS (kernel) version, number of processors and memory
  usage statistics (see
  [mach_host.defs](https://github.com/darlinghq/darling/blob/master/platform-include/mach/mach_host.defs)).
  Get it using `mach_host_self()`. There also exists a "privileged host control
  port" (`host_priv_t`) that allows privileged tasks (aka processes running as
  root) to *control* the host (see
  [host_priv.defs](https://github.com/darlinghq/darling/blob/master/platform-include/mach/host_priv.defs)).
  The official way to get it is by calling `host_get_host_priv_port()` passing
  the "regular" host port; in reality it returns either the same port name (if
  the task is privileged) or `MACH_PORT_NULL` (if it's not).

* **Task name port**, an Apple extension, an unprivileged version of the *task
  port*. It references the task, but does not allow controlling it. The only
  thing that seems to be available through it is `task_info()`.

* **Bootstrap port**, a send right to the *bootstrap server* (on Darwin, this is
  launchd). The bootstrap server serves as a *server registry*, allowing other
  servers to export their ports under well-known reverse-DNS names (such as
  `com.apple.system.notification_center`), and other tasks to look up those
  ports by these names. The bootstrap server is the primary way to "connect" to
  another task, comparable to D-Bus under Linux. The bootstrap port for the
  current task is available in the `bootstrap_port` global variable. On the
  Hurd, the filesystem serves as the server registry, and they use the bootstrap
  port for passing context to *translators* instead.

* **Seatbelt port**, **task access port**, **debug control port**, which are yet
  to be documented.

In addition to using the `*_self()` traps and other methods mentioned above, you
can get all these ports by calling `task_get_special_port()`, passing in the
task port (for the caller task or any other task) and an identifier of the port
(TASK_BOOTSTRAP_PORT` and so on). There also exist wrapper macros
(`task_get_bootstrap_port()` and so on) that pass the right identifier
automatically.

There also exists `task_set_special_port()` (and the wrapper macros) that allows
you to *change* the special ports for a given task to any send right you
provide. `mach_task_self()` and all the other APIs discussed above will, in
fact, return these replaced ports rather than the *real* ports for the task,
host and so on. This is a powerful mechanism that can be used, for example, to
disallow task to manipulate itself, log and forward any messages it sends and
receives (Hurd's `rpctrace`), or make it believe it's running on a different
version of the kernel or on another host. This is also how tasks get the
bootstrap port: the first task (launchd) starts with a null bootstrap port,
allocates a port and sets it as the bootstrap port for the tasks it spawns.

There also are two special sets of ports that tasks inherit:

* **Registered ports**, up to 3 of them. They can be registered using
  `mach_ports_register()` and later looked up using `mach_ports_lookup()`.
  Apple's XPC uses these to pass down its "XPC bootstrap port".
* **Exception ports**, which are the ports where the kernel should *send* info
  about *exceptions* happening during the task execution (for example, Unix's
  Segmentation Fault corresponds to `EXC_BAD_ACCESS`). This way, a task can
  handle its own exceptions or let another task handle its exceptions. This is
  how the Crash Reporter.app works on macOS, and this is also what LLDB uses.
  Under Darling, the Linux kernel delivers these kinds of events as Unix signals
  to the process that they happen in, then the process converts the received
  signals to Mach exceptions and sends them to the correct exception port (see
  [sigexc.c](https://github.com/darlinghq/darling/blob/master/src/kernel/emulation/linux/signal/sigexc.c)).

As a Darwin extension, there are `pid_for_task()`, `task_for_pid()`, and
`task_name_for_pid()` syscalls that allow converting between Mach task ports and
Unix PIDs. Since they essentially circumvent the capability model (PIDs are just
integers in the global namespace), they are crippled on iOS/macOS with UID
checks, entitlements, SIP, etc. limiting their use. On Darling, they are
unrestricted.

Similarly to task ports, threads have their corresponding **thread ports**,
obtainable with `mach_thread_self()` and interposable with
`thread_set_special_port()`.

## Higher-level APIs

Most programs don't construct Mach messages manually and don't call `mach_msg()`
directly. Instead, they use higher-level APIs or tools that wrap Mach messaging.

### MIG

MIG stands for **Mach Interface Generator**. It takes interface definitions like
this example:

```defs
// File: window.defs

subsystem window 35000;

#include <mach/std_types.defs>

routine create_window(
    server: mach_port_t;
    out window: mach_port_t);

routine window_set_frame(
    window: mach_port_t;
    x: int;
    y: int;
    width: int;
    height: int);
```

And generates the C boilerplate for serializing and deserializing the calls and
sending/handling the Mach messages. Here's a (much simplified) client code that
it generates:

```c
// File: windowUser.c

/* Routine create_window */
kern_return_t create_window(mach_port_t server, mach_port_t *window) {

	typedef struct {
		mach_msg_header_t Head;
	} Request;

	typedef struct {
		mach_msg_header_t Head;
		/* start of the kernel processed data */
		mach_msg_body_t msgh_body;
		mach_msg_port_descriptor_t window;
		/* end of the kernel processed data */
		mach_msg_trailer_t trailer;
	} Reply;

	union {
		Request In;
		Reply Out;
	} Mess;

	Request *InP = &Mess.In;
	Reply *Out0P = &Mess.Out;

	mach_msg_return_t msg_result;

	InP->Head.msgh_bits = MACH_MSGH_BITS(MACH_MSG_TYPE_COPY_SEND, MACH_MSG_TYPE_MAKE_SEND_ONCE);
	/* msgh_size passed as argument */
	InP->Head.msgh_request_port = server;
	InP->Head.msgh_reply_port = mig_get_reply_port();
	InP->Head.msgh_id = 35000;
	InP->Head.msgh_reserved = 0;

	msg_result = mach_msg(
		&InP->Head,
		MACH_SEND_MSG|MACH_RCV_MSG|MACH_MSG_OPTION_NONE, 
		(mach_msg_size_t) sizeof(Request),
		(mach_msg_size_t) sizeof(Reply),
		InP->Head.msgh_reply_port,
		MACH_MSG_TIMEOUT_NONE,
		MACH_PORT_NULL
	);

	if (msg_result != MACH_MSG_SUCCESS) {
		return msg_result;
	}

	*window = Out0P->window.name;
	return KERN_SUCCESS;
}

/* Routine window_set_frame */
kern_return_t window_set_frame(mach_port_t window, int x, int y, int width, int height) {

	typedef struct {
		mach_msg_header_t Head;
		int x;
		int y;
		int width;
		int height;
	} Request;

	typedef struct {
		mach_msg_header_t Head;
		mach_msg_trailer_t trailer;
	} Reply;

	union {
		Request In;
		Reply Out;
	} Mess;

	Request *InP = &Mess.In;
	Reply *Out0P = &Mess.Out;

	mach_msg_return_t msg_result;

	InP->x = x;
	InP->y = y;
	InP->width = width;
	InP->height = height;
	InP->Head.msgh_bits = MACH_MSGH_BITS(MACH_MSG_TYPE_COPY_SEND, MACH_MSG_TYPE_MAKE_SEND_ONCE);
	/* msgh_size passed as argument */
	InP->Head.msgh_request_port = window;
	InP->Head.msgh_reply_port = mig_get_reply_port();
	InP->Head.msgh_id = 35001;
	InP->Head.msgh_reserved = 0;

	msg_result = mach_msg(
		&InP->Head,
		MACH_SEND_MSG|MACH_RCV_MSG|MACH_MSG_OPTION_NONE,
		(mach_msg_size_t) sizeof(Request),
		(mach_msg_size_t) sizeof(Reply), 
		InP->Head.msgh_reply_port,
		MACH_MSG_TIMEOUT_NONE,
		MACH_PORT_NULL
	);

	if (msg_result != MACH_MSG_SUCCESS) {
		return msg_result;
	}

	return KERN_SUCCESS;
}
```

To get the reply from the server, MIG includes a port right (called the *reply
port*) in the message, and then performs a send on the server port and a receive
on the reply port with a single `mach_msg()` call. The client keeps the receive
right for the reply port, while the server gets sent a send-once right. This
way, event though MIG reuses a single (per-thread) reply port for all the
servers it talks to, servers can't impersonate each other.

And the corresponding server code, also simplified:

```c
// File: windowServer.c

/* Routine create_window */
extern kern_return_t create_window(mach_port_t server, mach_port_t *window);

/* Routine create_window */
mig_internal novalue _Xcreate_window(mach_msg_header_t *InHeadP, mach_msg_header_t *OutHeadP) {

	typedef struct {
		mach_msg_header_t Head;
		mach_msg_trailer_t trailer;
	} Request;

	typedef struct {
		mach_msg_header_t Head;
		/* start of the kernel processed data */
		mach_msg_body_t msgh_body;
		mach_msg_port_descriptor_t window;
		/* end of the kernel processed data */
	} Reply;

	Request *In0P = (Request *) InHeadP;
	Reply *OutP = (Reply *) OutHeadP;


	kern_return_t RetCode;

	OutP->window.disposition = MACH_MSG_TYPE_COPY_SEND;
	OutP->window.pad1 = 0;
	OutP->window.pad2 = 0;
	OutP->window.type = MACH_MSG_PORT_DESCRIPTOR;

	RetCode = create_window(In0P->Head.msgh_request_port, &OutP->window.name);

	if (RetCode != KERN_SUCCESS) {
		MIG_RETURN_ERROR(OutP, RetCode);
	}

	OutP->Head.msgh_bits |= MACH_MSGH_BITS_COMPLEX;
	OutP->Head.msgh_size = (mach_msg_size_t) (sizeof(Reply));
	OutP->msgh_body.msgh_descriptor_count = 1;
}


/* Routine window_set_frame */
extern kern_return_t window_set_frame(mach_port_t window, int x, int y, int width, int height);

/* Routine window_set_frame */
mig_internal novalue _Xwindow_set_frame(mach_msg_header_t *InHeadP, mach_msg_header_t *OutHeadP) {

	typedef struct {
		mach_msg_header_t Head;
		int x;
		int y;
		int width;
		int height;
		mach_msg_trailer_t trailer;
	} Request;

	typedef struct {
		mach_msg_header_t Head;
	} Reply;

	Request *In0P = (Request *) InHeadP;
	Reply *OutP = (Reply *) OutHeadP;

	window_set_frame(In0P->Head.msgh_request_port, In0P->x, In0P->y, In0P->width, In0P->height);
}


/* Description of this subsystem, for use in direct RPC */
const struct window_subsystem {
	mach_msg_id_t	start;		/* Min routine number */
	mach_msg_id_t	end;		/* Max routine number + 1 */
        unsigned int    maxsize;        /* Max msg size */
	struct routine_descriptor	/* Array of routine descriptors */
		routine[2];
} window_subsystem = {
	35000,
	35002,
	(mach_msg_size_t) sizeof(union __ReplyUnion__window_subsystem),
	{
		_Xcreate_window,
		_Xwindow_set_frame
	}
};

boolean_t window_server(mach_msg_header_t *InHeadP, mach_msg_header_t *OutHeadP) {
	mig_routine_t routine;

	OutHeadP->msgh_bits = MACH_MSGH_BITS(MACH_MSGH_BITS_REPLY(InHeadP->msgh_bits), 0);
	OutHeadP->msgh_remote_port = InHeadP->msgh_reply_port;
	/* Minimal size: routine() will update it if different */
	OutHeadP->msgh_size = (mach_msg_size_t) sizeof(mig_reply_error_t);
	OutHeadP->msgh_local_port = MACH_PORT_NULL;
	OutHeadP->msgh_id = InHeadP->msgh_id + 100;
	OutHeadP->msgh_reserved = 0;

	if ((InHeadP->msgh_id > 35001) || (InHeadP->msgh_id < 35000)) {
		return FALSE;
	}
	routine = window_subsystem.routine[InHeadP->msgh_id - 35000];
	(*routine) (InHeadP, OutHeadP);
	return TRUE;
}
```

Client-side usage looks just like invoking the routines:

```c
mach_port_t server_port = ...;
mach_port_t window;
kern_return_t kr;

kr = create_window(server_port, &window);

kr = window_set_frame(window, 50, 55, 200, 100);
```

And on the server side, you implement the corresponding functions and then call `mach_msg_server()`:

```c
kern_return_t create_window(mach_port_t server, mach_port_t *window) {
    // ...
}

kern_return_t window_set_frame(mach_port_t window, int x, int y, int width, int height) {
    // ...
}

int main() {
    mach_port_t server_port = ...;
    mach_msg_server(window_server, window_subsystem.maxsize, server_port, 0);
}
```

MIG supports a bunch of useful options and features. It's extensively used in
Mach for RPC (remote procedure calls), including for communicating between the
kernel and the userspace. Other than a few direct Mach traps such as
`msg_send()` itself, Mach kernel API functions (such as the ones for task and
port manipulation) are in fact MIG routines.

### Distributed Objects

Distributed Objects is a *dynamic* RPC platform, as opposed to MIG, which is
fundamentally based on static code generation. Distributed objects allows you to
transparently use [Objective-C](documentation/objective-c) objects from other
processes as if they were regular Objective-C objects.

```objc
// File: server.m

NSConnection *connection = [NSConnection defaultConnection];
[connection setRootObject: myAwesomeObject];
[connection registerName: @"org.darlinghq.example"];
[[NSRunLoop currentRunLoop] run];
```

```objc
// File: client.m

NSConnection *connection =
    [NSConnection connectionWithRegisteredName: @"org.darlinghq.example"
                                          host: nil];

MyAwesomeObject *proxy = [[connection rootProxy] retain];
[proxy someMethod];
```

There is no need to statically generate any code for this, it all works at
runtime through the magic of objc message forwarding infrastructure. Methods you
call may return other objects (or take them as arguments), and the distributed
objects support code will automatically either send a copy of the object to the
remote process (using `NSCoding`) or proxy methods called on those objects as
well.

### XPC

XPC is a newer IPC framework from Apple, tightly integrated with launchd. Its
lower-level C API allows processes to exchange plist-like data via Mach
messages. Higher-level Objective-C API (`NSXPC*`) exports a proxying interface
similar to Distributed Objects. Unlike Distributed Objects, it's asynchronous,
doesn't try to hide the possibility of connection errors, and only allows
passing whitelisted types (to prevent certain kinds of attacks).

Apple's XPC is not open source. On Darling, the low-level XPC implementation
([libxpc](https://github.com/darlinghq/darling-libxpc)) is based on NextBSD
libxpc. The high-level Cocoa APIs are not yet implemented.

## Useful resources

Note that Apple's version of Mach as used in XNU/Darwin is subtly different than
both OSF Mach and GNU Mach.

* [Inter Process Communication - The GNU Mach Reference Manual](https://www.gnu.org/software/hurd/gnumach-doc/Inter-Process-Communication.html)
* [Mach Kernel Interface Reference Manual](http://web.mit.edu/darwin/src/modules/xnu/osfmk/man/)
* [Changes to XNU Mach IPC](https://robert.sesek.com/2014/1/changes_to_xnu_mach_ipc.html)
* [Debugging Mach Ports](https://robert.sesek.com/2012/1/debugging_mach_ports.html)
* [Some Fun with Mach Ports](http://www.foldr.org/~michaelw/log/computers/macosx/task-info-fun-with-mach)
* [Reaching the MACH layer](http://blog.wuntee.sexy/reaching-the-mach-layer)
* [Interprocess communication on iOS with Mach messages](http://ddeville.me/2015/02/interprocess-communication-on-ios-with-mach-messages)
* [Mach Message and Bootstrap Server on OS X](http://przhu.github.io/using%20mac/2012/08/25/mach-message-and-bootstrap-server-on-os-x/)
* [Friday Q&A 2013-01-11: Mach Exception Handlers](https://www.mikeash.com/pyblog/friday-qa-2013-01-11-mach-exception-handlers.html)
* [Mach 3 Server Writer's Guide](http://shakthimaan.com/downloads/hurd/server_writer.pdf)
* [Revisiting Apple IPC: (1) Distributed Objects](https://googleprojectzero.blogspot.com/2015/09/revisiting-apple-ipc-1-distributed_28.html)
* [About Distributed Objects](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/DistrObjects/Concepts/AboutDistributedObjects.html)
* [GNUstep Distributed Objects](http://www.gnustep.it/nicola/Tutorials/DistributedObjects/)
* [Objective-C GNUstep Base Programming Manual: 7. Distributed Objects](http://www.gnustep.org/resources/documentation/Developer/Base/ProgrammingManual/manual_7.html)
* [Creating XPC Services](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingXPCServices.html)
* [Auditing and Exploiting Apple IPC](https://thecyberwire.com/events/docs/IanBeer_JSS_Slides.pdf)
