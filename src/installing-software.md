# Installing software

There are multiple ways to install software on macOS, and our aim is to make all of them work on Darling as well. However there currently are a few limitations, mainly the lack of GUI.

## You might not even need to install it

[Unlike Wine](https///wiki.winehq.org/FAQ#I_have_lots_of_applications_already_installed_in_Windows._How_do_I_run_them_in_Wine.3F), Darling can run software that's installed on an **existing macOS installation** on the same computer. This is possible thanks to the way application bundles (`.app`-s) work on macOS and Darling.

To use an app that's already installed, you just need to locate the existing installation (e.g. `/Volumes/SystemRoot/run/media/username/Macintosh HD/Applications/SomeApp.app`) and run the app from there.

## DMG files

Many apps for macOS are distributed as `.dmg` (disk image) files that contain the `.app` bundle inside. Under macOS, you would click the DMG to *mount* it and then drag the `.app` to your `Applications` folder to copy it there.

Under Darling, use `hdiutil attach SomeApp.dmg` to mount the DMG (the same command works on macOS too), and then copy the `.app` using `cp`:

```
Darling [~]$ hdiutil attach Downloads/SomeApp.dmg
/Volumes/SomeApp
Darling [~]$ cp -r /Volumes/SomeApp/SomeApp.app /Applications/
```

## Archives

Some apps are distributed as archives instead of disk images. To install such an app, unpack the archive using the appropriate CLI tools and copy the `.app` to `/Applications`.

## Mac App Store

Many apps are only available via Apple's Mac App Store. To install such an application in Darling, download the app from a real App Store (possibly running on another computer) and copy the `.app` over to Darling.

## PKG files

Many apps use `.pkg`, the native package format of macOS, as their distribution format. It's not enough to simply copy the contents of a package somewhere, they are really meant to be *installed* and can run arbitrary scripts during installation.

Under macOS, you would use the graphical Installer.app or the command-line `installer` utility to install this kind of packages. You can do the latter under Darling as well:

```
Darling [~]$ installer -pkg mc-4.8.7-0.pkg -target /
```

Unlike macOS, Darling also has the `uninstaller` command, which lets you easily uninstall packages.

## Package managers

There are many third-party package managers for macOS, the most popular one being [Homebrew](https///brew.sh/). Ultimately, we want to make it possible to use all the existing package managers with Darling, however, some may not work well right now.

## Command-line developer tools

To install command-line developer tools such as the C compiler (Clang) and LLDB, you can install Xcode
using one of the method mentioned above, and then run

```
Darling [~]$ xcode-select --switch /Applications/Xcode.app
```

Alternatively, you can download and install only command-line tools from Apple by running

```
Darling [~]$ xcode-select --install
```

Note that both Xcode and command-line tools are subject to Apple's EULA.
