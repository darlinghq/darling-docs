# Duct-taping XNU code

Part of what makes Darling's kernel module so unique is the fact that we use XNU code in Linux kernel-space in order to be as accurate to XNU's
functionality as possible, as well as to make it easier to replicate XNU functionality (because we don't have to rewrite all of it ourselves).
But how is the kernel module able to do this? Well, a big part of this is that when XNU code that we want to use needs certain other XNU code that we can't use,
we have to reimplement the functionality ourselves. How that works depends on the functionality needed.
However, using XNU code in the LKM requires more than just reimplementing certain functions.

## Pre- and post-XNU includes

Because Darling's kernel module is, well, a kernel module, it's built in the context of the Linux kernel with the Linux kernel build system.
This means that many preprocessor definitions which conflict with XNU's own definitions are used when compiling XNU files.
Additionally, XNU uses many macros and inline functions to provide functionality that we can't use and must replace with our own.

The solution to all of this? Pre- and post-XNU duct-tape headers. What this means it that, before any XNU headers are included in a source file,
we include the pre-XNU duct-tape header. Then, after all XNU headers are included in a source file, we include the post-XNU duct-tape header.

The pre-XNU duct-tape header makes sure that:
  * No conflicting Linux preprocessor definitions are present
  * XNU function names don't conflict with Linux ones
  * Some types that XNU headers want are defined to the ones we want

The post-XNU duct-tape header makes sure that:
  * Certain XNU macros and functions (like `copyin`/`copyout` or spinlock operations) are replaced with our own

## Modifying XNU code

Often, we can use *parts* of some XNU code, but not the whole thing. In that case, we wrap the unwanted code in an `#ifndef __DARLING__` block.
In other cases, we can use some XNU code with some minor modifications. In these cases, we have to modify the original code.
In order to clearly mark our modifications, we wrap them in `#ifdef __DARLING__` blocks and leave the original XNU code in `#else` blocks.
If the function would require *many* modifications, it's best to reimplement it instead.

## Stubs

In reality, many seemingly required functions can just be stubbed and the code that depends on them will still work! For examples, just check out the [duct directory](https://github.com/darlinghq/darling-newlkm/tree/master/duct).
Reasons may vary&mdash;maybe the code using them will fall back to a different codepath or maybe they're never actually called&mdash;but in any case,
we can just stub them and get away with it.
