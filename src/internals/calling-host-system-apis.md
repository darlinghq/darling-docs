# Calling host system APIs

This article describes how Darling enables code compiled into Mach-O files to
interact with API exported from ELF files on the host system. This is needed,
for example, to access ALSA audio APIs.

Apple's dynamic loader (dyld) cannot load ELF files and extending it in this
direction would be a rather extensive endeavor, also because Apple's linker
(ld64) would need to be extended at the same time. This means some kind of
bridge between the host platform's ELF loader and the "Mach-O world" has to be
set up.

## The ELF Bridge

mldr is responsible for [loading Mach-O's](basics/loader.md). However, it is also responsible for providing the ELF
bridge for the Mach-O world. Because it is an ELF itself, it has full access to a normal Linux ELF environment,
complete with dynamic library loading and pthread functionality (which is necessary for
[Darling's threading implementation](threading/thread-implementation.md)).
How does mldr allow the Mach-O world to use this stuff? As part of loading a Mach-O binary,
mldr populates a structure with the addresses of ELF functions we want to use.
After populating this structure ([`struct elf_calls`](https://github.com/darlinghq/darling/blob/master/src/startup/mldr/elfcalls/elfcalls.h#L17)),
it passes it to the Mach-O binary as part of the special `applep` environment array.
Mach-O code can then call those functions at any time using the function pointers stored in the structure.

## Wrappers

To enable easy linking, a concept of ELF wrappers was introduced, along with a
tool named `wrapgen`. `wrapgen` parses ELF libraries, extracts the SONAME (name
of library to be loaded) and a list of visible symbols exported from the
library.

Now that we have the symbols, a neat trick is used to make them available to
Mach-O applications. The Mach-O format supports so called *symbol resolvers*,
which are functions that return the address of the symbol they represent.
`dyld` calls them and provides the result as symbol address to whoever needs
the symbol.

Therefore, `wrapgen` produces C files such as this:

```c	
#include <elfcalls.h>
extern struct elf_calls* _elfcalls;

static void* lib_handle;
__attribute__((constructor)) static void initializer() {
    lib_handle = _elfcalls->dlopen_fatal("libasound.so.2");
}

__attribute__((destructor)) static void destructor() {
    _elfcalls->dlclose_fatal(lib_handle);
}

void* snd_pcm_open() {
    __asm__(".symbol_resolver _snd_pcm_open");
    return _elfcalls->dlsym_fatal(lib_handle, "snd_pcm_open");
}
```

The C file is then compiled into a Mach-O library, which transparently wraps an
ELF library on the host system.

## CMake integration

To make things very easy, there is a CMake function that automatically takes
care of wrapping a host system's library.

Example:

```cmake
include(wrap_elf)
include(darling_exe)

wrap_elf(asound libasound.so)

add_darling_executable(pcm_min pcm_min.c)
target_link_libraries(pcm_min system asound)
```

The `wrap_elf()` call creates a Mach-O library of given name, wrapping an ELF
library of given name, and installs it into `/usr/lib/native` inside the
prefix.
