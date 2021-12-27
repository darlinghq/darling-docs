# Generating Syscalls

While not common, there are situations where you might need to regenerate syscalls. Fortunately, Apple provides a convenient perl script called `create-syscalls.pl`. The script is located in `[XNU]/libsyscall/xcodescripts/`.

## Understanding `create-syscalls.pl`'s Arugments

When you try to run the command, without any arguments, you will get the following prompt:

```
Usage: create-syscalls.pl syscalls.master custom-directory platforms-directory platform-name out-directory
```

Unfortunately, the prompt does not give a lot of helpful information. Below is a breakdown of what the script is requesting.

* `syscalls.master` - Point to the syscalls.master file. ex: `[XNU]/bsd/kern/syscalls.master`
* `custom-directory` - Point to a directory that contains a `SYS.h`, `custom.s`, and a list of `__` files. These files can be found in a folder called custom. ex:`[XNU]/libsyscall/custom/`
* `platforms-directory`- Point to a directory that contains the `syscall.map` files. Ex:`[XNU]/libsyscall/Platforms/`
* `platform-name` - One of the platform names that are in the platform directory. ex: `MacOSX`

In addition, you will need to define the list of `ARCHS` that the tool should generate.
```
export ARCHS="arm64 i386 x86_64"
```

## Example

```
export ARCHS="arm64 i386 x86_64"
mkdir ~/Desktop/generated_syscalls
./create-syscalls.pl $XNU/bsd/kern/syscalls.master $XNU/libsyscall/custom/ $XNU/libsyscall/Platforms/ MacOSX ~/Desktop/generated_syscalls
```
