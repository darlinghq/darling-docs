# Cross Compilation

Darling can be built on one architecture for another. This page describes a
cross build from an x86_64 Debian host to arm64. The same approach applies to
other distributions that provide an aarch64 toolchain and target libraries.

A cross build produces two kinds of executables:

* Darling's Linux-side runtime (`mldr`, `darlingserver`, the launcher) and the
  generated Darwin/Mach-O binaries are built for the target architecture.
* A small set of build tools (`ld64`, `ar`, `lipo`, `install_name_tool`,
  `adhoc-sign`, `rtsig`, and others) are built for the build machine, because
  they have to run while Darling is being built. These are expected to be
  x86_64 even though the Darling installation is arm64.

## Installing arm64 libraries on Debian with multiarch

Enable the arm64 architecture and update the package lists:

```bash
sudo dpkg --add-architecture arm64
sudo apt update
```

Install the cross toolchain. Clang uses the GNU cross toolchain's startup
files, C library headers, and C++ standard library headers when targeting
aarch64:

```bash
sudo apt install crossbuild-essential-arm64
```

Debian multiarch packages are selected with the `:arm64` suffix. Install the
target variants of the build dependencies. The following list covers the
dependencies from the Debian 13 native build instructions; install only what
the components you are building need if you want a smaller set:

```bash
sudo apt install \
  cmake clang bison flex xz-utils pkg-config libcap2-bin git git-lfs \
  libc6-dev:arm64 libssl-dev:arm64 libxml2-dev:arm64 libdbus-1-dev:arm64 \
  libfuse-dev:arm64 libudev-dev:arm64 libbsd-dev:arm64 libcap-dev:arm64 \
  libpulse-dev:arm64 libavformat-dev:arm64 libavcodec-dev:arm64 \
  libswresample-dev:arm64 libvulkan-dev:arm64 libcurl4-openssl-dev:arm64 \
  libedit-dev:arm64 libxkbfile-dev:arm64 libcairo2-dev:arm64 \
  libglu1-mesa-dev:arm64 libgl1-mesa-dev:arm64 libtiff-dev:arm64 \
  libfreetype-dev:arm64 libfontconfig1-dev:arm64 libegl1-mesa-dev:arm64 \
  libgif-dev:arm64 libxrandr-dev:arm64 libxcursor-dev:arm64
```

## Configuring the build

Configure the build with Clang targeting the GNU aarch64 triplet. Both the C
and C++ compiler targets must be set, and `CMAKE_SYSTEM_PROCESSOR` selects the
architectures Darling builds:

```bash
mkdir build-arm64
cd build-arm64

env PKG_CONFIG_LIBDIR=/usr/lib/aarch64-linux-gnu/pkgconfig:/usr/share/pkgconfig \
cmake .. \
  -DCOMPONENTS=cli \
  -DCMAKE_INSTALL_PREFIX="$HOME/darling" \
  -DCMAKE_SYSTEM_PROCESSOR=aarch64 \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DCMAKE_C_COMPILER_TARGET=aarch64-linux-gnu \
  -DCMAKE_CXX_COMPILER_TARGET=aarch64-linux-gnu \
  -DCMAKE_ASM_COMPILER_TARGET=aarch64-linux-gnu
```

`PKG_CONFIG_LIBDIR` makes `pkg-config` read the target's `.pc` files. Without
it, CMake and `pkg-config` can find the host's x86_64 libraries and the
Linux-side executables will fail to link with errors such as:

```text
libcrypto.so: error adding symbols: file in wrong format
```

When `CMAKE_SYSTEM_PROCESSOR` is an arm64 name, the build enables
`TARGET_ARM64` and disables the x86 and i386 target slices automatically. The
32-bit build is not attempted.

## Building and installing

```bash
cmake --build . -j"$(nproc)"
cmake --install .
```

If the installation prefix is only writable by root, run the install step
with `sudo`. If you are installing into a directory you own and the host has
no `sudo`, tell the install-time shutdown helper that no privileged escalation
is needed:

```bash
DARLING_NONROOT=1 cmake --install .
```

After installation, the Linux-side executables (`bin/darling`, `bin/darpath`,
`bin/darlingserver`, `mldr`) should be AArch64 ELF files, and the Darwin
binaries under `libexec/darling` should be Mach-O arm64 or arm64-only
universal binaries.
