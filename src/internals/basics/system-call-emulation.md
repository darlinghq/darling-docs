# System call emulation

System calls are the most critical interface of all user space software. They
are the means of invoking kernel's functionality. Without them, software would
not be able to communicate with the user, access the file system or establish
network connections.

Every kernel provides a very specific set of system calls, even if there is a
similar set of libc APIs available on different systems. For example, the system
call set greatly differs between Linux and FreeBSD.

MacOS uses a kernel named XNU. XNU's system calls differ greatly from those of
Linux, which is why XNU system call emulation is at the heart of Darling.

## XNU system calls

Unlike other kernels, XNU has three distinct system call tables:

 1. **BSD system calls**, invoked via sysenter/syscall. These calls frequently
    have a direct counterpart on Linux, which makes them easiest to implement.
 2. **Mach system calls**, which use negative system call numbers. These are
    non-existent on Linux and are mostly implemented in darlingserver.
    These calls are built around [Mach ports](../macos-specifics/mach-ports.md).
 3. **Machine dependent system calls**, invoked via `int 0x82`. There are only a
    few of them and they are mainly related to [thread-local
    storage](../threading/thread-local-storage.md).

## Introduction

Darling emulates system calls by providing a modified
`/usr/lib/system/libsystem_kernel.dylib` library, the source code of which is
located in `src/kernel`. However, some parts of the emulation are located in
Darling's userspace kernel server (a.k.a. darlingserver, located in `src/external/darlingserver`).

This is why `libsystem_kernel.dylib` (and also `dyld`, which contains a static
build of this library) can never be copied from macOS to Darling.

Emulating XNU system calls directly in Linux would have a few benefits, but
isn't really workable. Unlike BSD kernels, Linux has no support for foreign
system call emulation and having such an extensive and intrusive patchset merged
into Linux would be too difficult. Requiring Darling's users to patch their
kernels is out of question as well.

In the past, Darling used a kernel module to implement many of the same things now handled by darlingserver.
This approach had certain advantages (e.g. much simpler to manage Darling processes and typically faster than darlingserver),
but it was found to be too unstable. Additionally, debugging kernel code is much harder than debugging userspace code, so it also presented
development challenges.

### Disadvantages of this approach

* Inability to run a full copy of macOS under Darling (notwithstanding the
  legality of such endeavor), at least the files mentioned above need to be
  different.

* Inability to run software making direct system calls. This includes some old UPX-packed executables. In the past, software written in Go also belonged in this group, but this is no longer the case. Note that [Apple provides no
  support](https://developer.apple.com/library/content/qa/qa1118/_index.html)
  for making direct system calls (which is effectively very similar to
  distributing statically linked executables described in the article) and
  frequently changes the system call table, hence such software is bound to
  break over time.

### Advantages of this approach

* Significantly easier development.
* No need to worry about having patches merged into Linux.
* It is easier for users to have newest code available, it is not needed to run
  the latest kernel to have the most up to date system call emulation.

## Implementation

libsystem_kernel consists of two main components: Apple's open-source wrappers and our own syscall emulation.
This setup makes our emulation model what happens on macOS as closely as possible. Apple uses wrappers in order to perform
additional set up for syscalls as necessary and this also allows them to modify the kernel syscall interface between releases
while keeping a stable libsystem_kernel interface.

When the wrappers are called, they perform any necessary set up and then invoke the syscall using a macro.
This macro normally expands to a bit of assembly code that includes a `syscall` instruction, but for Darling, this macro is replaced
with our own modified version that redirects the call over to one of our syscall lookup functions (according to the syscall type: BSD, Mach, or machdep).
Our handler then looks in the appropriate table and invokes our emulated version of the syscall (or returns `ENOSYS` if we don't support it).

After that, the process becomes largely syscall-specific and documenting how we handle each and every one of them here would be a large task, and probably
not that useful (after all, if you're looking to dive into the specifics of libsystem_kernel, it's probably best to look at the code). However,
it *is* useful to document details common across multiple syscalls, so that is what follows now.

### Making Linux syscalls

The majority of our emulated libsystem_kernel syscalls need to make Linux syscalls (and some need to make multiple syscalls).
In order to make this easier, libsystem_kernel has a `LINUX_SYSCALL` macro. It can be used like so: `LINUX_SYSCALL(linux_syscall_number, arguments...)`.
As an example, this is how our `chroot` uses it:

```c
long sys_chroot(const char* path) {
  // ... other code ...
  LINUX_SYSCALL(__NR_chroot, path);
  // ... more code ...
};
```

In addition, to avoid dependence on external headers, libsystem_kernel includes copies of Linux headers defining syscall numbers. All of these syscall numbers
are prefixed with `__NR_`. The syscall numbers, as well as availability, depends on the architecture. See [this directory](https://github.com/darlinghq/darling/tree/master/src/kernel/emulation/linux/linux-syscalls) for the header we include.

### Making darlingserver calls

Many syscalls can be emulated using only Linux syscalls. However, there are some, most notably Mach syscalls, that require calls into our userspace kernel server.

darlingserver calls have wrappers in libsystem_kernel that handle all the RPC boilerplate for messaging the server.
Prototypes for these wrappers can be included using `#include <darlingserver/rpc.h>`. A full list of
all the supported RPC calls can be found [here](https://github.com/darlinghq/darlingserver/blob/main/scripts/generate-rpc-wrappers.py#L22).

Each wrapper is of the form:

```c
int dserver_rpc_CALL_NAME_HERE(int arg1, char arg2 /* and arg3, arg4, etc. */, int* return_val1, char* return_val2 /* and return_val3, return_val4, etc. */);
```

The arguments to the wrapper are the arguments (if any) to the RPC call followed by pointers to store the return values (if any) from the call.
All return value pointers can be set to `NULL` if you don't want/care about that value.

File descriptors can also be sent and received like any other argument or return value; the wrappers take care of the necessary RPC boilerplate behind
the scenes.

For example, this is the wrapper prototype for the `uidgid` call, which retrieves the UID and GID of the current process (as seen within the process)
and optionally updates them (if either `new_uid` or `new_gid` are greater than `-1`):

```c
int dserver_rpc_uidgid(int32_t new_uid, int32_t new_gid, int32_t* out_old_uid, int32_t* out_old_gid);
```

The following are all valid invocations of this wrapper:

```c
int old_uid;
int old_gid;

// retrieve the old UID and set the new UID to 5 (without changing the GID)
dserver_rpc_uidgid(5, -1, &old_uid, NULL);

// retrieve the old UID and GID and don't change the UID or GID values
dserver_rpc_uidgid(-1, -1, &old_uid, &old_gid);

// set the new GID to 89 (without changing the UID)
dserver_rpc_uidgid(-1, 89, NULL, NULL);
```

For more details on darlingserver RPC, see the [appropriate doc](../darlingserver/rpc.md).

### `errno` translation

Every syscall needs to return a status code, and Linux syscalls also return status codes. Therefore, many of our emulated syscalls that make Linux syscalls
return the Linux syscall's return code as their own. However, since `errno` values differ between BSD and Linux, we need to translate these codes before returning them.
That's what `errno_linux_to_bsd` is for: feed it a Linux `errno` and it'll give you the equivalent BSD `errno`.
It will also automatically handle signs: if you give a positive code, it'll return a positive code, and vice versa for negative codes.

### "Simple" implementations of certain libc functions

Libc provides some incredibly useful functions that we simply cannot use in libsystem_kernel,
due the fact that libsystem_kernel can't have any external dependencies.
However, we would still like to have many of these available in libsystem_kernel. Solution? Implement our own simple versions of these functions.

The result is that everything in [`simple.h`](https://github.com/darlinghq/darling/blob/master/src/kernel/emulation/linux/simple.h) is available in libsystem_kernel.
These are all designed to provide the necessary functionality from their libc counterparts. This means, for example,
that most (but not all) format specifiers are supported by `__simple_printf`, mainly just because we don't need them all.
Additionally, `__simple_kprintf` is libsystem_kernel-unique: it can be used to print to the Linux kernel log stream (using our LKM).

### vchroot handling

As part of Darling's [containerization](containerization.md), Darling uses something called a virtual chroot, or "vchroot" for short.
The idea behind this technique is that, since we control libsystem_kernel, through which all Darwin filesystem operations must go,
we can rewrite paths in libsystem_kernel to make the root (i.e. `/`) point to our prefix (e.g. `~/.darling`). The main advantage
of this method compared to an actual chroot is that Linux libraries and programs used within the prefix are oblivious to this and see the normal Linux root.
This means that Linux libraries can communicate with daemons and services exactly like they normally would, requiring no extra setup on our part.

The disadvantage of this approach is that means a lot more work for our libsystem_kernel. We have to be careful to translate every single path
we handle, both ones we receive as input (i.e. we must expand them to their full Linux paths) and ones we return (i.e. we must unexpand them back to their prefix-relative paths).
Thankfully, there are two functions in libsystem_kernel that handle all the expansion and unexpansion to make it easier on syscalls: `vchroot_expand` and `vchroot_unexpand`.

`vchroot_expand` expects a pointer to an argument structure (for legacy reasons; this used to be an LKM call). This structure contains 3 members.
On input, `path` is a character buffer containing the path to expand. On output, it contains the expanded path.
`flags` specifies flags for the call. Currently, there is only a single flag: `VCHROOT_FOLLOW` tells the function to return the expanded symlink target path if the input path refers to a symlink.
`dfd` specifies a descriptor pointing to a directory to use as the base from which to resolve relative input paths. If it is set to `-100`, the process's current working directory.
`vchroot_expand` returns `0` on success or a Linux errno (remember to [translate this](#errno-translation) if returning it from a syscall!).

`vchroot_unexpand` does the opposite of `vchroot_expand`: it takes a fully expanded Linux path, and returns the unexpanded prefix-relative path.
Like `vchroot_expand`, it expects a pointer to an argument structure, although it only has a single member: `path`.
On input, `path` is a character buffer containing the path to unexpand. On output, `path` contains the unexpanded path.
Like `vchroot_expand`, `vchroot_unexpand` returns `0` on success or a Linux errno.
