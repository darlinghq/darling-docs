# Additional Update Guidelines for ICU

As a reminder, these are *guidelines*. The version you update to might make some of these steps obsolete, so make sure you understand what they do and why they're necessary.

## Generate `icudtXXl.dat.xz` Artifact

The following commands should be executed on Linux.

1. Create and set up the build directory

```sh
mkdir build
cp -rp icuSources/* build/
cd build
chmod +x runConfigureICU && chmod +x configure
```

2. Patch `tools/icuinfo/icuinfo.cpp` to remove references to Mach components. 

Example patch for `tools/icuinfo/icuinfo.cpp`:
```patch
--- ../icuSources/tools/icuinfo/icuinfo.cpp	2019-09-07 22:44:27.000000000 -0400
+++ tools/icuinfo/icuinfo.cpp	2020-07-01 10:21:00.796333708 -0400
@@ -308,7 +308,9 @@

 // Apple addition
 #include <unistd.h>
+#if 0
 #include <mach/mach_time.h>
+#endif
 #include <unicode/ustring.h>
 #include <unicode/udat.h>
 enum { kUCharsOutMax = 128, kBytesOutMax = 256 };
@@ -319,15 +321,18 @@
     static const UDate udatTry1 = 1290714600000.0; // 2010 Nov. 25 (Thurs) 11:50:00 AM PT
     static const UDate udatTry2 = 1451736016000.0; // 2016 Jan. 02  ...
     int remaining = 2;
+#if 0
     mach_timebase_info_data_t info;
     mach_timebase_info(&info);
+#endif
     while (remaining-- > 0) {
+#if 0
         uint64_t start, durationOpen, durationUse1, durationUse2;
         UDateFormat *udatfmt;
         int32_t datlen1, datlen2;
         UChar outUChars[kUCharsOutMax];
         UErrorCode status = U_ZERO_ERROR;
-
+
         start = mach_absolute_time();
         udatfmt = udat_open(UDAT_MEDIUM, UDAT_FULL, locale, tzName, -1, NULL, 0, &status);
         durationOpen = ((mach_absolute_time() - start) * info.numer)/info.denom;
@@ -347,8 +352,11 @@
             }
             udat_close(udatfmt);
         } else {
+#endif
             printf("first time %d udat_open failed\n", remaining);
+#if 0
         }
+#endif
     }
 }
```

3. Set up the build flags and execute `runConfigureICU`.
```sh
APPLE_INTERNAL_DIR="/AppleInternal" \
LANG="en_US.utf8" \
CPPFLAGS="-DU_DISABLE_RENAMING=1"  \
CFLAGS="-DU_SHOW_CPLUSPLUS_API=1 -DU_SHOW_INTERNAL_API=1 -DICU_DATA_DIR=\"\\\"/usr/local/share/icu/\\\"\" -DICU_DATA_DIR_PREFIX_ENV_VAR=\"\\\"APPLE_FRAMEWORKS_ROOT\\\"\" -m64 -g -Os -fno-exceptions -fvisibility=hidden" \
CXXFLAGS="-std=c++11 -DU_SHOW_CPLUSPLUS_API=1 -DU_SHOW_INTERNAL_API=1 -DICU_DATA_DIR=\"\\\"/usr/local/share/icu/\\\"\" -DICU_DATA_DIR_PREFIX_ENV_VAR=\"\\\"APPLE_FRAMEWORKS_ROOT\\\"\" -m64 -g -Os -fno-exceptions -fvisibility=hidden -fvisibility-inlines-hidden" \
DYLD_LIBRARY_PATH="/usr/local/lib" \
./runConfigureICU Linux --disable-renaming --disable-extras --disable-layout --disable-samples --with-data-packaging=archive --prefix=/usr/local --with-library-bits=64
```

4. In a new terminal window, set up the build flags and run `make`
```sh
APPLE_INTERNAL_DIR="/AppleInternal" \
LANG="en_US.utf8" \
CFLAGS="-DU_SHOW_CPLUSPLUS_API=1 -DU_SHOW_INTERNAL_API=1 -DICU_DATA_DIR=\"\\\"/usr/local/share/icu/\\\"\" -DICU_DATA_DIR_PREFIX_ENV_VAR=\"\\\"APPLE_FRAMEWORKS_ROOT\\\"\" -m64 -g -Os -fno-exceptions -fvisibility=hidden" \
CXXFLAGS="-std=c++11 -DU_SHOW_CPLUSPLUS_API=1 -DU_SHOW_INTERNAL_API=1 -DICU_DATA_DIR=\"\\\"/usr/local/share/icu/\\\"\" -DICU_DATA_DIR_PREFIX_ENV_VAR=\"\\\"APPLE_FRAMEWORKS_ROOT\\\"\" -m64 -g -Os -fno-exceptions -fvisibility=hidden -fvisibility-inlines-hidden" \
DYLD_LIBRARY_PATH="/usr/local/lib" \
make
```

5. Save and compress the generated `icudtXXl.dat` artifact

The version number might not be the same as before. If that is the case, you will need to update the `CMakeLists.txt` as well to use the new `icudtXXl.dat` file.

```sh
# For this example, the version number provided is 64
mkdir ../icuSources/data/out
cp data/out/icudt64l.dat ../icuSources/data/out/icudt64l.dat
xz -z ../icuSources/data/out/icudt64l.dat
```

6. Remove the build folder

```sh
cd ..
rm -rf build
```