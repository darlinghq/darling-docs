# Packaging

**NOTE:** This is not extensivly tested, and may break

## Debian

To package Darling for Debian-baded system we provide the ```tools/makedeb``` script.

All output files are stored in ```..``` because of a technical limtation of ```debuild```.

### Install Dependencies
```bash
$ sudp apt install devscripts equivs dpkg-dev
```

### Building Binary Packages

#### Install Build Dependencies
```bash
sudo mk-build-deps -ir
```

#### Build
```bash
$ tools/makedeb
```

### Building Sources Packages

Use this if you want to upload to a service like Launchpad.

```bash
$ tools/makedeb --dsc
```

## RPM

TBD
