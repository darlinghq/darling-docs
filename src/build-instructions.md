# Build Instructions

You must be running a 64-bit x86 Linux distribution. Darling cannot be used on a 32-bit x86 system, not even to run 32-bit applications.

# Dependencies

Clang is required to compile Darling; at least Clang 6 is required. You can force a specific version of Clang (if it is installed on your system) by editing `Toolchain.cmake`.

A minimum of 4 GB of RAM is also required for building. Using swap space may help reduce the memory usage, but is likely to slow the build down significantly.

Linux 5.0 or higher is required.


**Debian 10/11**

```bash
sudo apt install cmake clang-6.0 bison flex xz-utils libfuse-dev libudev-dev pkg-config \
libc6-dev-i386 libcap2-bin git git-lfs python2 libglu1-mesa-dev libcairo2-dev \
libgl1-mesa-dev libtiff5-dev libfreetype6-dev libxml2-dev libegl1-mesa-dev libfontconfig1-dev \
libbsd-dev libxrandr-dev libxcursor-dev libgif-dev libpulse-dev libavformat-dev libavcodec-dev \
libswresample-dev libdbus-1-dev libxkbfile-dev libssl-dev llvm-dev
```

**Debian Testing**

```bash
sudo apt install cmake clang-9 bison flex xz-utils libfuse-dev libudev-dev pkg-config \
libc6-dev-i386 libcap2-bin git git-lfs python2 libglu1-mesa-dev libcairo2-dev \
libgl1-mesa-dev libtiff5-dev libfreetype6-dev libxml2-dev libegl1-mesa-dev libfontconfig1-dev \
libbsd-dev libxrandr-dev libxcursor-dev libgif-dev libpulse-dev libavformat-dev libavcodec-dev \
libswresample-dev libdbus-1-dev libxkbfile-dev libssl-dev llvm-dev
```

**Ubuntu 18.04/20.04:**

```bash
sudo apt install cmake clang bison flex libfuse-dev libudev-dev pkg-config libc6-dev-i386 \
gcc-multilib libcairo2-dev libgl1-mesa-dev libglu1-mesa-dev libtiff5-dev \
libfreetype6-dev git git-lfs libelf-dev libxml2-dev libegl1-mesa-dev libfontconfig1-dev \
libbsd-dev libxrandr-dev libxcursor-dev libgif-dev libavutil-dev libpulse-dev \
libavformat-dev libavcodec-dev libswresample-dev libdbus-1-dev libxkbfile-dev \
libssl-dev python2
```

**Arch Linux & Manjaro:**

Due to the Python 2 dependency, an AUR is needed. If you don't have `yay` already, install it:

```bash
sudo pacman -S --needed git base-devel
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

Install dependencies:

```bash
yay -S python2
sudo pacman -S --needed make cmake clang flex bison icu fuse gcc-multilib \
lib32-gcc-libs pkg-config fontconfig cairo libtiff mesa glu llvm libbsd libxkbfile \
libxcursor libxext libxkbcommon libxrandr ffmpeg git git-lfs
```

**Fedora and CentOS**

[RPMFusion](https://rpmfusion.org/RPM%20Fusion) is required for FFmpeg.

```bash
sudo dnf install make cmake clang bison dbus-devel flex python2 glibc-devel.i686 fuse-devel \
systemd-devel elfutils-libelf-devel cairo-devel freetype-devel.{x86_64,i686} \
libjpeg-turbo-devel.{x86_64,i686} libtiff-devel.{x86_64,i686} fontconfig-devel.{x86_64,i686} \
libglvnd-devel.{x86_64,i686} mesa-libGL-devel.{x86_64,i686} mesa-libEGL-devel.{x86_64,i686} \
mesa-libGLU-devel.{x86_64,i686} libxml2-devel libbsd-devel git git-lfs libXcursor-devel \
libXrandr-devel giflib-devel ffmpeg-devel pulseaudio-libs-devel libxkbfile-devel \
openssl-devel llvm libcap-devel
```

**OpenSUSE Tumbleweed**

You will need to build Darling with only the 64bit components. See **Build Options** for instructions. 

```bash
sudo zypper install make cmake-full clang10 bison flex python-base glibc fuse-devel \
libsystemd0 libelf1 cairo-devel libfreetype6 libjpeg-turbo libfontconfig1 libglvnd \
Mesa-libGL-devel Mesa-libEGL-devel libGLU1 libxml2-tools libbsd-devel git git-lfs \
libXcursor-devel giflib-devel ffmpeg-4 ffmpeg-4-libavcodec-devel \
ffmpeg-4-libavformat-devel libpulse-devel pulseaudio-utils libxkbfile-devel openssl \
llvm libcap-progs libtiff-devel libjpeg8-devel libXrandr-devel dbus-1-devel glu-devel \
ffmpeg-4-libswresample-devel
```

**Alpine Linux**

Alpine Linux doesn't ship Python 2 any more which currently is required to build Darling.
The oldest release that still does is Alpine Linux 3.15 so the following instructions are only for that version.

Make sure to [enable the community repository](https://wiki.alpinelinux.org/wiki/Enable_Community_Repository).
Alpine also doesn't support 32-bit builds, so make sure to [disable that](#disabling-32-bit-libraries).

```bash
sudo apk add cmake clang bison flex xz fuse-dev pkgconfig libcap git git-lfs python2 \
python3 glu-dev cairo-dev mesa-dev tiff-dev freetype-dev libxml2-dev fontconfig-dev \
libbsd-dev libxrandr-dev libxcursor-dev giflib-dev pulseaudio-dev ffmpeg-dev dbus-dev \
libxkbfile-dev openssl-dev libexecinfo-dev make gcc g++ xdg-user-dirs
```

These are the minimum requirements for building and running Darling on Alpine.
Of course, if you want to run GUI applications, you'll also need a desktop environment.

# Fetch the Sources

Darling uses git-lfs. Set this up if needed with [official instructions](https://github.com/git-lfs/git-lfs/wiki/Installation).

Darling makes extensive use of Git submodules, therefore you cannot use a plain `git clone`. Make a clone like this:

```bash
git clone --recursive https://github.com/darlinghq/darling.git
```

**Attention:** The source tree requires up to 5 GB of disk space!

# Updating sources

If you have already cloned Darling and would like to get the latest changes, do this in the source root:

```bash
git lfs install
git pull
git submodule update --init --recursive
```

# Build

The build system of Darling is CMake. Makefiles are generated by CMake by default.

**Attention:** The build may require up to 16 GB of disk space! The Darling installation itself then takes up to 1 GB.

## Building and Installing

Now let's build Darling:

```bash
# Move into the cloned sources
cd darling

# Make a build directory
mkdir build && cd build

# Configure the build
cmake ..

# Build and install Darling
make
sudo make install
```

## Build Options

### Disabling 32-bit Libraries

Darling normally builds both 32-bit and 64-bit versions of all libraries, to enable 32-bit programs to run under Darling.
However, this means Darling also requires 32-bit version of certain native libraries. If you can't setup a multilib environment or you just
want to build only the 64-bit components, use `-DTARGET_i386=OFF` during configuration to disable building the 32-bit components.

### Parallel Builds

Another way to speed up the build is to run `make` with multiple jobs. For this, run `make -j8` instead, where 8 is a number of current jobs to run of your choosing. In general, avoid running more jobs than twice the amount CPU cores of your machine.

### "Unified" JavaScriptCore Builds

If you still want to build JavaScriptCore and have a bit of RAM to spare, JavaScriptCore also supports a build mode known as ["unified builds"](https://blogs.gnome.org/mcatanzaro/2018/02/17/on-compiling-webkit-now-twice-as-fast/). This build mode can cut JSC build times in half, at the expense of causing slightly higher RAM usage. This build mode can be enabled in Darling by adding `-DJSC_UNIFIED_BUILD=ON` when configuring the build.

### Ninja build system

As an alternative to `make`, `ninja` comes with parallelism on by default and a nicer progress indicator.

Follow the normal build instructions until you get to the `cmake ..` step. Replace that with `cmake .. -GNinja`. Now you can build with `ninja` instead of `make`.

#### Ninja fails to link libcsu

If you are using Ninja, the library "libcsu" might fail to link. The solution is to remove the library located at "./src/external/csu/libcsu.a" and try again. See [this issue](https://github.com/darlinghq/darling/issues/915) for more information.

### Debug Builds

By default, CMake setups up a non-debug, non-release build.
If you run LLDB and encounter messages indicating a lack of debug symbols, make sure you are doing a debug build. To do this, use `-DCMAKE_BUILD_TYPE=Debug`.

### Unit tests

Darling has a limited number of unit tests. These are not currently built by default, but this can be enabled with '-DENABLE_TESTS=1'.
These tests are then installed to /usr/libexec within your Darling container.

### Additional, Non-standard Binaries

Darling tries to stick to a standard macOS installation as much as possible. However, if you would like to build and install some additional packages (such as GNU tar), you can add `-DADDITIONAL_PACKAGES=ON`.

### Custom Installation Prefix

To install Darling in a custom directory, use the ``CMAKE_INSTALL_PREFIX`` CMake option. However, a Darling installation is **NOT** portable, because the installation prefix is hardcoded into the ``darling`` executable. This is intentional. If you do move your Darling installation, you will get this error message:
```
Cannot mount overlay: No such file or directory
Cannot open mnt namespace file: No such file or directory
```
If you wish to properly move your Darling installation, the only supported option is for you to uninstall your current Darling installation, and then rebuild Darling with a different installation prefix.

### Building Only Certain Components

By default, almost all of Darling is built, similar to what would be a full macOS installation. However, you can also choose to only build certain components of Darling with the `COMPONENTS` configuration option in CMake. This is a comma-separated list of components to build. By default, `COMPONENTS` is set to `stock` (which includes various other components). The following components are currently recognized:

  * Basic components
    * `core` - Builds darlingserver, mldr, launchd, and libSystem (in addition to a couple of host-side executables). This is the minimal set of targets necessary to run an executable under Darling. Note that most executables depend on various other programs and system services which are not provided by this component; this will only support extremely bare-bones executables (e.g. a "Hello world" executable).
    * `system` - Includes `core` and everything necessary to enter a shell (including a shell: bash and zsh) and perform basic system functions (i.e. essential system daemons).
  * Script runtimes - Note that you'll likely also want to include `system` with these, but it's not strictly required
    * `python` - Includes `core` and the Python 2.7 runtime and standard library, along with additional programs that require Python.
    * `ruby` - Includes `core` and the Ruby 2.6 runtime and standard library, along with additional programs that require Ruby.
    * `perl` - Includes `core` and the Perl 5.18 and 5.28 runtimes and standard libraries, along with additional programs that require Perl.
  * CLI components
    * `cli` - Includes `system` and most standalone command-line programs (i.e. those that don't require additional runtimes or complex frameworks/libraries).
      * Note that this does *not* include Vim; this is because Vim depends on AppKit and CoreServices (which are considered GUI components). The `cli_dev` component, however, does include those frameworks, so it also includes Vim.
    * `cli_dev` - Includes `cli`, `python`, `ruby`, and `perl`, along with some additional targets normally regarded as GUI targets. These additional targets are necessary for certain parts of the Xcode CLI tools. This is the component you want if you want to build and develop software with Xcode on the command line and don't need/want the full GUI installation.
    * `cli_extra` - Includes `cli` and some additional programs not installed on a standard macOS installation (e.g. GNU tar).
  * GUI components - Note that none of these components include `cli` or any other CLI-related components. Some apps may rely on certain CLI programs or script runtimes being available, so you may also need to include `cli`, `python`, `ruby`, and more yourself.
    * `gui` - Includes `system` and everything necessary to run basic Cocoa, Metal, and OpenGL apps. Note that only includes the minimum required for GUI applications; most applications will also require additional frameworks (e.g. audio, video, location, etc.).
    * `gui_frameworks` - Includes `gui` and many GUI-related frameworks depended upon by Cocoa apps (e.g. audio frameworks). Note that this component only includes non-stub frameworks.
    * `gui_stubs` - Includes `gui_frameworks` and many stubbed GUI-related frameworks.
  * Heavy frameworks - These are frameworks that can significantly slow down the build and/or take up a lot of space after installation but are unnecessary for the vast majority of programs.
    * `jsc` - JavaScriptCore - This is a large framework used for executing JavaScript code, primarily in web browsers. This typically delays the build by an extra 30 minutes to 1 hour.
    * `webkit` - WebKit - This is another large framework that is only used for web browsers (embedded or otherwise).
  * Umbrella components - These are components that only include other components, intended to make it easy to select a large swath of components for common builds.
    * `stock` - Includes `cli`, `python`, `ruby`, `perl`, and `gui_stubs` (these implicitly include `core`, `system`, `gui`, and `gui_frameworks`). This is the default component selected for standard builds and is intended to match the software included by default with a standard macOS installation.
    * `all` - Includes every single other component. In other words, this includes `stock`, `jsc`, `webkit`, and `cli_extra`.

## Known Issues

### BackBox

If your distribution is Backbox and you run into build issues try the following commands:

```bash
sudo update-alternatives --install /usr/bin/clang clang /usr/bin/clang-6.0 600
sudo update-alternatives --install /usr/bin/clang++ clang++ /usr/bin/clang++-6.0 600
```

### SELinux

On SELinux you may see the following error when starting Darling:

```
Cannot open mnt namespace file: No such file or directory
```

To work around this try this command: `setsebool -P mmap_low_allowed 1`.

### Broken Symbolic Link

Darling relies heavily on symbolic links. It is important to build Darling on a filesystem that supports this feature.

If you are still running into issues, despite downloading and building Darling on a filesystem that supports symbolic links, check your git configuration to make sure that you have not intentionally disabled it (ex: `core.symlinks=false`).

### File System Support

Darling uses overlayfs for implementing prefixes on top of the macOS-like root filesystem. While overlayfs is not very picky about the lower (read-only) filesystem (where your `/usr` lives), it has stricter requirements for the upper filesystem (your home directory, unless you override the `DPREFIX` environment variable).

To quote the [kernel documentation](https://www.kernel.org/doc/Documentation/filesystems/overlayfs.txt):

> The lower filesystem can be any filesystem supported by Linux and does not need to be writable. The lower filesystem can even be another overlayfs. The upper filesystem will normally be writable and if it is it must support the creation of trusted.* extended attributes, and must provide valid d_type in readdir responses, so NFS is not suitable.

In addition to NFS not being supported, ZFS and eCryptfs encrypted storage are also known not to work.

If you try to use an unsupported file system, this error will be printed:

```
Cannot mount overlay: Invalid argument
```
