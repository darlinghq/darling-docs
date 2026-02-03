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
