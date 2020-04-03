# Thread-Local Storage

Thread-Local Storage (TLS) is a critical feature enabling store and retrieval of per-thread data.

In user applications (written in C/C++), TLS variables are typically accessed via ''pthread_getspecific()'' and ''pthread_setspecific()'' or via special attributes ''__thread'' or ''thread_local'' (since C++11). However, TLS is no less important even in applications that do not make use of this facility explicitly.

This is because TLS is a much needed functionality used by the threading library libpthread (otherwise ''pthread_self()'' would not work) and also in libc itself (''errno'' is typically a per-thread value to avoid races).

On 32/64-bit x86, old [segment registers](https///en.wikipedia.org/wiki/X86_memory_segmentation) from the 16-bit times have found new use thanks to TLS.

## TLS Setup

 1.  When a new thread is being set up, a block of memory is allocated by libpthread. This block of memory is usually what ''pthread_self()'' returns.
 2.  libpthread asks the kernel to set up a new [GDT](https///en.wikibooks.org/wiki/X86_Assembly/Global_Descriptor_Table) entry that refers to this memory block.
 3.  The entry number is then set into the respective segment register.

### Linux Specific

On Linux, there are two different system calls used to create GDT entries.


*  ''set_thread_area()'' is available on both x86 and x86-64.

*  ''arch_prctl()'' is only available on x86-64, but unlike ''set_thread_area()'' it also supports 64-bit base addresses.

### macOS Specific

A machine-dependent ("machdep") system call ''thread_fast_set_cthread_self()'' is used to create a new segment. Darling translates this call to one of the above Linux system calls. 

## TLS Usage

The concept of memory segments is very simple. You can imagine that the segment refers to a certain area of your process' virtual memory. Any memory accesses you make via the segment register are relative to start of that area and are limited by area's size.

Imagine that you have set up FS to point to an integer array. Now you can set an element whose index is in EAX to value 42:

	
	movl $42, %fs:(%eax, 4)


## Registers

While x86 offers a total of six segment registers, only two of them (FS and GS) can be used for TLS (the others being CS, DS, ES and SS). This effectively limits the number of facilities managing TLS areas in a single process to two. One of them is typically used by the platform's native libc, whilst the other one is typically used by foreign libc's (courtesy of Darling and Wine).

The following table explains which registers are used by whom:

 | System                      | TLS register on i386 | TLS register on x86-64 | 
 | ------                      | -------------------- | ---------------------- | 
 | Linux libc                  | GS                   | FS                     | 
 | Apple's libSystem           | GS                   | GS                     | 
 | Apple's libSystem (Darling) | FS                   | GS                     | 

This is also why Wine for macOS can never run under Darling: it would always overwrite either the register used by Linux libc or Darling's libSystem.


