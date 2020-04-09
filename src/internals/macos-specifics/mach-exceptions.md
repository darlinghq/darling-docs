# Mach Exceptions

This section documents how unix signals and Mach exceptions interact within XNU.

Unlike XNU, Linux uses unix signals for all purposes, i.e. for both typically hardware-generated (e.g. `SIGSEGV`) and software-generated (e.g. `SIGINT`) signals.

## Hardware Exceptions

In XNU, hardware (CPU) exceptions are handled by the Mach part of the kernel (`exception_triage()` in `osfmk/kern/exception.c`). As a result of such an exception, a Mach exception is generated and delivered to the exception port of given task.

By default - i.e. when the debugger hasn't replaced the task's exception ports - these Mach exceptions make their way into `bsd/uxkern/ux_exception.c`, which contains a kernel thread receiving and translating Mach exceptions into BSD signals (via `ux_exception()`). This translation takes into account hardware specifics, also by calling `machine_exception()` in `bsd/dev/i386/unix_signal.c`. Finally, the BSD signal is sent using `threadsignal()` in `bsd/kern/kern_sig.c`.

## Software Signals

Software signal processing in XNU follows the usual pattern from BSD/Linux.

However, if `ptrace(PT_ATTACHEXC)` was called on the target process, the signal is also translated into a Mach exception via `do_bsdexception()`, the pending signal is removed and the process is set to a stopped state (`SSTOP`).

The debugger then has to call `ptrace(PT_THUPDATE)` to set the unix signal number to be delivered to the target process (which could also be `0`, i.e. no signal) and that also resumes the process (state `SRUN`).


## Debugging under Darling

Debugging support in Darling makes use of what we call "cooperative debugging".
It means the code in the debuggee is aware it's being debugged and actively
assists the process. In Darling, this role is taken on mainly by
[sigexc.c](https://github.com/darlinghq/darling/blob/master/src/kernel/emulation/linux/signal/sigexc.c)
in `libsystem_kernel.dylib`, so no application modifications are necessary.

MacOS debuggers use a combination of BSD-like and Mach APIs to control and
inspect the debuggee.

To emulate the macOS behavior, Darling makes use of POSIX real-time signals to
invoke actions in the cooperative debugging code.

| Operation | macOS | Linux | Darling implementation |
| --------- | ----- | ----- | ---------------------- |
| Attach to debuggee | `ptrace(PT_ATTACHEXC)` <br> Causes the kernel to redirect all signals (aka exceptions) to the Mach "exception port" of the process. Only debuggee termination is notified via `wait()`. | `ptrace(PTRACE_ATTACH)` <br> Signals sent to the debuggee and the debuggee termination event are received in the debugger via `wait()`. | Notify the LKM that we will be tracing the process. Send a RT signal to the debuggee to notify it of the situation. The debuggee sets up handlers to handle all signals and forward them to the exception port. |
| Examine registers | `thread_get_state(X86_THREAD_STATE)` | `ptrace(PTRACE_GETREGS)` | Upon receiving a signal, the debuggee reads its own register state and passes it to the kernel via `thread_set_state()`. |
| Pausing the debuggee | `kill(SIGSTOP)` | `kill(SIGSTOP)` or `ptrace(PTRACE_INTERRUPT)` | Send a RT signal to the debuggee that it should act as if `SIGSTOP` were sent to the process. We cannot send a real `SIGSTOP`, because then the debuggee couldn't provide/update register state to the debugger etc. |
| Change signal delivery | `ptrace(PT_THUPDATE)` | `ptrace(PTRACE_rest)` | Send a RT signal to the debuggee to inform it what it should do with the signal (ignore, pass it to the application etc.) |
| Set memory watchpoints | `thread_set_state(X86_DEBUG_STATE)` | `ptrace(PTRACE_POKEUSER)` | Implement the effects of `PTRACE_POKEUSER` in the LKM. |
