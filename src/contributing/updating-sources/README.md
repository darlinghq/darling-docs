# Updating Sources

Darling contains many components built from open source Apple code, such as libdispatch, libc, Security, and many more.

Every so often, Apple releases updated versions of these components on their [open source portal](https://opensource.apple.com/) (typically shortly after a new OS update).

This section is a guide for any developers that find themselves needing to update Darling's copies of Apple's open source components.

## General Steps

Each project is different, but for the most part, these are the steps you should follow. Note that these are general guidelines; most projects require additional steps before they'll build/run correctly!

### <a name="step-1"></a>1. Replace the current source with the updated source

The first step should be to download the updated source and replace the current source with it. You should first delete all the files in the current source ***except the `CMakeLists.txt`, `darling` folder, and `README.md` file***, then copy over the updated source.

Most Apple sources don't contain a `CMakeLists.txt` / `README.md` of their own, but if one does, you should delete it or (preferably) rename it to something else (e.g. `CMakeLists.apple.txt` / `README.apple.md`).

### <a name="step-2"></a>2. Create an initial update commit

You should now create a commit that includes all the changes (e.g. `git add -A`) with a message containing the name of the project and the version it was updated to (e.g. `Libc-1353.60.8`). This is done to clearly separate our changes from Apple's original code and it makes it easier to see this distinction in the Git history.

### <a name="step-3"></a>3. Update the `CMakeLists.txt`

The next step is to update the `CMakeLists.txt` for Darling to actually build it. Generally, the `CMakeLists.txt` won't need to be changed much, except for maybe adding or removing files that were added or removed in the new source. In case the new source does need more modifications in order to build, you can usually refer to the Xcode build information in the project (`*.xcodeproj`, `*.xcconfig`, etc.) to determine what flags or sources need to be included.

### <a name="step-4"></a>4. Review the Git history for the project

You should check the previous source files (usually with Git) to see if any Darling-specific modifications were made to the code. If so, review the modifications to see whether they're still necessary for the updated code. All modifications are normally marked as being Darling-specific somehow. See the next step for usual markers for Darling-specific changes.

### <a name="step-5"></a>5. Make source modifications if necessary

Most of Apple's code builds just fine, and when problems do arise, more often than not, a change in compiler flags or definitions in the `CMakeLists.txt` will resolve the problem. Nonetheless, there are cases where Darling-specific workarounds are required. In these cases, you should try to keep your modifications to a minimum and only use them as a last resort in case all other methods of trying to fix the problem fail (e.g. check if any files are missing; they might need to be generated; maybe there's a script that needs to be run).

If you make modifications to the code, mark your changes as Darling-specific somehow. This serves as a way to identify our changes and makes it easier to update projects in the future. The way to do so depends on what kind of source file you're modifying, but here's a list of ways for a couple of languages:

  * C/C++/Objective-C/Objective-C++/MIG
    ```c
    #ifdef DARLING
    // Darling-specific modifications...
    #endif // DARLING
    ```

    or...

    ```c
    #if defined(DARLING) && OTHER_CONDITIONS
    // Darling-specific modifications...
    #endif
    ```
  * Shell scripts
    ```sh
    ## DARLING
    # Darling-specific modifications...
    ## /DARLING
    ```

### <a name="step-6"></a>6. Build and test it

This might seem obvious but this is the most important step. Before proposing any changes to Darling, make sure they at least build!

Most Darling subprojects don't have any real test suites, so "testing" here means making sure that the changes don't break something that was previously working. For now, that means testing out various applications and programs inside a Darling environment built with your changes.

While these kinds of checks will still be performed by the Darling team before accepting the changes, it is still highly recommended that you test your changes yourself and save everybody some time.

### <a name="step-7"></a>7. Commit your final changes

Finally, to propose your changes to be merged into Darling, commit your changes, preferably with a message that indicates that it contains Darling-specific modifications for the project and optionally what changes you made.

## Additional Notes

Like it was mentioned earlier, most projects require additional modifications and tweaks to work. For the subprojects that require additional steps, you can find the instructions in `[SUBMODULE]/darling/notes/UPDATE_SOURCE.md` (ex: [`darling-security/darling/notes/UPDATE_SOURCE.md`](https://github.com/darlinghq/darling-security/blob/master/darling/notes/UPDATE_SOURCE.md)).

Note that these instructions document what has had to be done until now; the upstream sources *could* completely switch up their setup from one version to the next, but until now, project structures have been pretty stable. Nonetheless, these are still *guidelines*; whenever sources are updated, you need to make sure to review them and perform any additional steps as necessary (and if possible, please document them).

### Generating Source Files From `.y`/`.l` Files

While working on some projects, you might encounter files like `dt_grammar.y`/`dt_lex.l`. These are `Yacc`/`Lex` source files. 

Normally, it is recommended to let XCode generate these sources files. However, you sometimes might run into issues building on XCode. The next alternative is to use `bison`/`flex` to manually generate these files.

```
bison -b -y -d ../../lib/libdtrace/common/dt_grammar.y
flex ../../lib/libdtrace/common/dt_lex.l
```
### Getting Make-Based Projects To Build

Out of the box, some projects that use a `MakeFile` don't build properly on macOS. This requires some additional paths to be added to `CPATH` and `LIBRARY_PATH`.

Option #1: Exclusively use the Darling SDK
```bash
export DARLING_PATH="/Volumes/CaseSensitive/darling"
export DARLING_SDK_PATH="$DARLING_PATH/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk"
export LIBRARY_PATH="$LD_LIBRARY_PATH:$DARLING_SDK_PATH/usr/lib"
export CPATH="$CPATH:$DARLING_SDK_PATH/usr/include:$DARLING_PATH/framework-include:$DARLING_PATH/framework-private-include"
```

Option #2: Use Apple's CLT & Darling SDK (for Frameworks)
```bash
export DARLING_PATH="/Volumes/CaseSensitive/darling"
export MACOS_CMDTOOLS_SDK_PATH="/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk"
export LIBRARY_PATH="$MACOS_CMDTOOLS_SDK_PATH/usr/lib"
export CPATH="$MACOS_CMDTOOLS_SDK_PATH/usr/include:$DARLING_PATH/framework-include:$DARLING_PATH/framework-private-include"
```

## Updating Version Number

### Updating SystemVersion Plist Files

The `SystemVersion.plist` contains information about the macOS (and iOS compatibility layer) version number. We keep this file in [`src/frameworks/CoreServices`](https://github.com/darlinghq/darling/tree/master/src/frameworks/CoreServices).

The recommended approach is to copy over the file from a real macOS install (Located in `/System/Library/CoreServices/`) and change `ProductBuildVersion` and `ProductCopyright` to `Darling` and `2012-[CURRENT YEAR] Lubos Dolezel`. 

Below is an example of macOS 11.7.4, with the changes applied.
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>ProductBuildVersion</key>
	<string>Darling</string>
	<key>ProductCopyright</key>
	<string>2012-2023 Lubos Dolezel</string>
	<key>ProductName</key>
	<string>macOS</string>
	<key>ProductUserVisibleVersion</key>
	<string>11.7.6</string>
	<key>ProductVersion</key>
	<string>11.7.6</string>
	<key>iOSSupportVersion</key>
	<string>14.7</string>
</dict>
</plist>
```

Next to the `SystemVersion.plist` file is `SystemVersionCompat.plist`. This file is used for older macOS programs that were not updated to handle the new version scheme introduced in Big Sur.

Below is an example of macOS 11.7.4, with the changes applied to `ProductBuildVersion` and `ProductCopyright`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>ProductBuildVersion</key>
	<string>Darling</string>
	<key>ProductCopyright</key>
	<string>2012-2023 Lubos Dolezel</string>
	<key>ProductName</key>
	<string>Mac OS X</string>
	<key>ProductUserVisibleVersion</key>
	<string>10.16</string>
	<key>ProductVersion</key>
	<string>10.16</string>
	<key>iOSSupportVersion</key>
	<string>14.7</string>
</dict>
</plist>
```

### Updating `kernel/emulation/linux/CMakeLists.txt` File

In this build file, there are variables that contain information about the kernel and system version.

#### Obtaining The Kernel Version

Use the command `uname -a` on macOS install you want to grab the kernel version from.

```zsh
% uname -a
Darwin Users-Mac 20.6.0 Darwin Kernel Version 20.6.0: Thu Mar  9 20:39:26 PST 2023; root:xnu-7195.141.49.700.6~1/RELEASE_X86_64 x86_64
```

Then copy and paste the values into `EMULATED_RELEASE` and `EMULATED_VERSION`.

```diff
 add_definitions(-DBSDTHREAD_WRAP_LINUX_PTHREAD
 	-DEMULATED_SYSNAME="Darwin"
- 	-DEMULATED_RELEASE="19.6.0"
- 	-DEMULATED_VERSION="Darwin Kernel Version 19.6.0"
+	-DEMULATED_RELEASE="20.6.0"
+	-DEMULATED_VERSION="Darwin Kernel Version 20.6.0"
 	-DEMULATED_OSVERSION="19G73"
 	-DEMULATED_OSPRODUCTVERSION="10.15.2"
)
```

#### Obtaining the System Version

The `EMULATED_OSVERSION` and `EMULATED_OSPRODUCTVERSION` values can be obtained from the `SystemVersion.plist` file. Refer to the **[Updating SystemVersion Plist Files](#updating-systemversion-plist-files)** for instructions on where to obtain this file.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>ProductBuildVersion</key>
	<string>20G1231</string>
	<key>ProductCopyright</key>
	<string>1983-2023 Apple Inc.</string>
	<key>ProductName</key>
	<string>macOS</string>
	<key>ProductUserVisibleVersion</key>
	<string>11.7.6</string>
	<key>ProductVersion</key>
	<string>11.7.6</string>
	<key>iOSSupportVersion</key>
	<string>14.7</string>
</dict>
</plist>
```

```diff
 add_definitions(-DBSDTHREAD_WRAP_LINUX_PTHREAD
 	-DEMULATED_SYSNAME="Darwin"
	-DEMULATED_RELEASE="20.6.0"
	-DEMULATED_VERSION="Darwin Kernel Version 20.6.0"
- 	-DEMULATED_OSVERSION="19G73"
- 	-DEMULATED_OSPRODUCTVERSION="10.15.2"
+ 	-DEMULATED_OSVERSION="20G1231"
+ 	-DEMULATED_OSPRODUCTVERSION="11.7.6"
)
```