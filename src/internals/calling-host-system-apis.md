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

The set of information provided to [dyld](basics/loader.md#dyld) by
[mldr](basics/loader.md#mldr) whenever a Mach-O executable is to be launched is
precisely defined. You probably know the typical C main function signature:

```c
int main(int argc, const char** argv)
```

If you have fiddled with this stuff before, you probably know there is also an
extra `const char** envp` parameter containing all of the environment variables.
Well, on Apple's systems, `envp` is followed by `const char** apple` containing
miscelaneous extra pieces of information, e.g. a full path to the executable
being run or possibly a block of random bytes as a seed for libc.

So this gives us the following signature:

```c
int main(int argc, const char** argv, const char** envp, const char** apple)
```

These arguments are passed to `main` by dyld, which receives them on the stack
from `mldr` (or from the kernel on a real macOS). By the way, these arguments
are also passed to all initializers marked with `__attribute__((constructor))`.

The `apple` parameter is ideal for passing additional information without
interfering with the environment undesirably. `mldr`
[declares](https://github.com/darlinghq/darling/blob/master/src/startup/elfcalls.h)
`struct elf_calls` with a set of function pointers, instantiates it and fills it
out. An address of this structure is then added as a string formatted as
`elf_calls=%p` into `apple`.

The address of `struct elf_calls` is later parsed inside libsystem_kernel an
exported as a symbol named `_elfcalls`.

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
