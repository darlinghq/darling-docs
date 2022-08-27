# Packaging
**NOTE:** This is not extensively tested, and may break

## Debian
To package Darling for Debian-based systems, we provide the `tools/debian/makedeb` script.

All output files are stored in the parent directory of the source root because of a technical limitation of `debuild`.

### Install Dependencies
```
bash
sudo apt install devscripts equivs dpkg-dev debhelper
```

### Building Binary Packages

#### Install Build Dependencies
```
bash
sudo mk-build-deps -ir
```

#### Build
```
bash
tools/debian/make-deb
```

### Build Source Packages
Use this if you want to upload to a service like Launchpad.

```
bash
tools/debian/make-deb --dsc
```

## RPM

### Build
1. Install ``docker`` and ``docker-compose``
2. ``cd rpm``
3. Build the docker image: ``docker-compose build rpm``
3. Build the rpms: ``docker-compose run rpm`` (Can take over half an hour)
4. Now you can run ``dnf install RPMS/x84_64/darling*.rpm``
5. If using SELinux, run ``setsebool -P mmap_low_allowed 1`` to allow Darling low level access

### Build for other operating systems
By default, the package will be built for Fedora 30. To build for a different OS, simply use:
```
RPM_OS=fedora:31 docker-compose build rpm
```

### Future improvements
- Everything is based off of ``dnf``. Supporting ``zypper`` and ``yum`` will reach others
- Because of the way the submodules are handled, this isn't quite ready for official releasing but this can be solved using [%autosetup in the %prep to checkout the submodules.](https://fedoraproject.org/wiki/Packaging:SourceURL#Git_Submodules)
