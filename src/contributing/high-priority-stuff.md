# High priority stuff

The intention of this page is to serve as a location for pointing developers to
areas where work is most needed.

## Work to be done


* Reimplement [CoreCrypto](https://github.com/darlinghq/darling-corecrypto).

  CoreCrypto's source code is publicly available, but its license prevents us
  from using it for Darling. Luckily, it's not that difficult! Some work has
  already been done. Having CoreCrypto will also enable us to build a recent
  version of [CommonCrypto](https://github.com/darlinghq/darling-commoncrypto).

* AppKit.framework

  More details to be added.

* libxpc
  * Implement missing APIs - many things in `xpc_dictionary` are missing.
  * `xpc_main()` should access the calling application's bundle and automatically listen as the defined `XPCService`.
    * This is a little tricky, because libxpc may not use CoreFoundation to parse the `Info.plist` (at least not directly). It should use `_NSGetExecutablePath()` and `xpc_create_from_plist()`.

* Foundation.framework
  * Implement `NSXPC*` classes that wrap libxpc.

* CoreAudio.framework
  * Implement CoreAudio.framework as the core place for platform audio abstraction (PulseAudio, ALSA, ...).
  * Modify existing AudioUnit code to use CoreAudio instead of accessing PA/ALSA directly.
  * Implement AUGraph and AudioQueue utility APIs.

* CoreServices.framework
  * Implement LaunchServices APIs for applications and file type mappings, backed by a database.
  * Implement UTI (Uniform Type Identifiers) API, also backed by a database.

