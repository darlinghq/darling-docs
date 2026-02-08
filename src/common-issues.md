# Common Issues

This page provides help for common issues you may run into when running programs under Darling.

## Cannot mmap segment __TEXT at 0x1000: Operation not permitted

Example:

```
Darling [/Volumes/NetSurf-2.9/NetSurf.app/Contents/MacOS]$ ./NetSurf 
Cannot mmap segment __TEXT at 0x1000: Operation not permitted
```

This occurs when the program is trying to mmap memory at too low of a memory address than allowed by the Linux kernel.

You will need to adjust the `mmap_min_addr` kernel tunable, Debian provides a good overview [here](https://wiki.debian.org/mmap_min_addr).

You can temporarily set the mmap min address to zero with:

```
sudo sysctl -w vm.mmap_min_addr="0"
```

See the Debian link above for more details.

## SELinux

On SELinux you may see the following error when starting Darling:

```
Cannot open mnt namespace file: No such file or directory
```

To work around this try this command: 

```
setsebool -P mmap_low_allowed 1
```

## File System Support

Darling uses overlayfs for implementing prefixes on top of the macOS-like root filesystem. While overlayfs is not very picky about the lower (read-only) filesystem (where your `/usr` lives), it has stricter requirements for the upper filesystem (your home directory, unless you override the `DPREFIX` environment variable).

To quote the [kernel documentation](https://www.kernel.org/doc/Documentation/filesystems/overlayfs.txt):

> The lower filesystem can be any filesystem supported by Linux and does not need to be writable. The lower filesystem can even be another overlayfs. The upper filesystem will normally be writable and if it is it must support the creation of trusted.* extended attributes, and must provide valid d_type in readdir responses, so NFS is not suitable.

In addition to NFS not being supported, ZFS and eCryptfs encrypted storage are also known not to work. However, fscrypt encrypted storage has been tested to work.

If you try to use an unsupported file system, this error will be printed:

```
Cannot mount overlay: Invalid argument
```
