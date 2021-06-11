# Building for the WSL

The Windows Subsystem for Linux (or WSL for short) allows you to run Linux programs on a Windows system.
WSL 2 is an improvement over WSL 1 that can run a full Linux kernel, complete with module loading.
Darling isn't supported on WSL 1 due to the need for a kernel module, but it *is* supported in WSL 2.

However, Microsoft's standard WSL 2 kernels are incompatible with Darling, due the use of a particular configuration flag (`CONFIG_EMBEDDED`),
as well as the fact that they do not include an entry in `/lib/modules` and there are no kernel headers distributed for them.
Thankfully, Microsoft has added the ability to use custom kernels with WSL 2. This guide walks you through the process
of building a kernel for the WSL that you can use Darling with.

## 0. Things to keep in mind

When building Darling on the WSL, be sure to **perform all steps in a directory native to the WSL**. Cloning and attempting to build Darling in a mounted Windows directory, such as one under `/mnt/c`, will result in a build failure down the line, as directories that are non-native to the WSL have different treatment of file capitilization.

## 1. Future-proofing this guide

Microsoft updates the kernel versions in their repo, so to make these steps more future-proof to work across different versions, all of the shell
snippets provided in this guide will assume the following variable is set:

```sh
# update this according the current version in Microsoft's repo
export MAJOR_MINOR_VERSION=5.4
```

Make sure to double-check this value and set it to the appropriate version you want to build;
check the list of branches on [Microsoft's repo](https://github.com/microsoft/WSL2-Linux-Kernel) to see which versions are available.
Keep in mind that Darling requires a 5.x kernel.

## 2. Build the kernel

This script will clone the Microsoft kernel rules, perform the necessary configuration modifications, and then build the kernel.
If you'd like to speed up the build, add `-j <2 * cpu cores>` to the `make` invocation.

All these commands should be executed in a WSL shell.

```sh
# clone the kernel to `kernel`; and only clone the commit we want to build
git clone --depth 1 --single-branch --branch linux-msft-wsl-${MAJOR_MINOR_VERSION}.y 'https://github.com/microsoft/WSL2-Linux-Kernel.git' kernel

cd kernel

# disable CONFIG_EMBEDDED
sed -i 's/CONFIG_EMBEDDED=y/CONFIG_EMBEDDED=n/g' Microsoft/config-wsl

# build the kernel
# this is where you can speed up the build using the `-j` flag
make KCONFIG_CONFIG=Microsoft/config-wsl
```

## 3. Install the kernel into the WSL

In the same shell (with the same working directory), run these commands:

```sh
# install the modules directory
sudo make modules_install

# install the kernel
sudo make install
```

## 4. Copy the kernel to Windows

Now we need to copy the kernel to your Windows drive so that we can configure the WSL to use it.
Again, these steps should be performed in the same WSL shell as before (i.e. same working directory).

```sh
# create a directory on your C drive for easy access
mkdir -p /mnt/c/linux-kernels

# now, let's copy the kernel to your Windows drive
cp arch/x86_64/boot/bzImage /mnt/c/linux-kernels/darling-wsl-kernel
```

Note that the copy might fail if the file you're trying to overwrite is the kernel currently in use by the WSL.
In that case, you'll need to copy it to another name, shutdown the WSL, and then manually rename the file in Explorer.

You can of course just leave it with a different name, but then you'll need to modify the path used in step 6.

## 5. Shutdown the WSL

Let's shutdown the WSL so it can detect our changes later. Run this in a Windows shell (Powershell or CMD, it doesn't really matter).

```sh
wsl --shutdown
```

## 6. Tell Windows about the kernel

Now we need to tell Windows that we want to use a different WSL kernel than the default one.
We can use a [`.wslconfig`](https://docs.microsoft.com/en-us/windows/wsl/wsl-config#configure-global-options-with-wslconfig) to tell Windows about this.

Create the following `.wslconfig` in the root of your user home directory (or modify it, if you already have one there):

```ini
[wsl2]
kernel=C:\\linux-kernels\\darling-wsl-kernel
```

This is just this absolute Windows path to the kernel we want to use (with escaped backslashes).
If you used a different filename in step 4, remember to change it here.

## 7. Profit!

...or something along those lines. Now, when you open up a new WSL shell, it should be using the custom kernel. You can verify this by
running `uname -r` in a WSL shell. It should return something along the lines of `<some-version>-microsoft-standard-WSL2+`.
The `+` means it has been modified, which means we're successfully using our custom kernel!

Now, you can go ahead and [build Darling](build-instructions.md) normally. Have fun using Darwin in Linux in Windows!
