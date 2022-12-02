# Additional Update Guidelines for JavaScriptCore

As a reminder, these are *guidelines*. The version you update to might make some of these steps obsolete, so make sure you understand what they do and why they're necessary.

## Generate Derived & Unified Sources

The following commands are executed from Darling. For generating the derived sources, it is recommended to execute the command in a darling shell. However, you can get away with generating the unified sources on Linux. 

```bash
export DARLING_SOURCE="/path/to/darling/source"
export BUILT_PRODUCTS_DIR=$(pwd)
export BUILD_SCRIPTS_DIR="$DARLING_SOURCE/src/external/WTF/Scripts"
export SRCROOT=$(pwd)
export SDKROOT="$DARLING_SOURCE/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk"
Scripts/generate-derived-sources.sh
Scripts/generate-unified-sources.sh

# For some reason, the JavaScriptCore uses the absolute path, instead of a relative path. The following commands fix that issue.
rm DerivedSources/JavaScriptCore/JavaScriptCore
cd DerivedSources/JavaScriptCore && ln -s ../../ JavaScriptCore && cd ../..
```

When uploading the sources to GitHub, make sure to not include the `.pyc` files.

## Generate Offline Assembly

Some source code requires that the offline assembly headers are generated (ex: `LLIntAssembly.h`).

Before you execute the following commands, make sure to check/update the `generate-offlineasm.sh` script first.

```bash
export DARLING_SOURCE="/path/to/darling/source"
export DARLING_SDK_ROOT="$DARLING_SOURCE/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk"
export DARLING_BUILD_ROOT="$DARLING_SOURCE/build"
./generate-offlineasm.sh
```
