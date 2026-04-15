# High priority stuff

The intention of this page is to serve as a location for pointing developers to
areas where work is most needed.

## CoreCrypto

CoreCrypto's source code is publicly available, but its license prevents us
from using it for Darling. Luckily, it's not that difficult! Some work has
already been done.

## Cocotron

### AppKit

More details to be added.

## libxpc & launchd

  * Implement missing APIs
    * Most (if not all) of the missing APIs are private ones
  * Implement XPC domain support in launchd
    * This is required for proper XPC service support, since at the moment, only system-wide services (i.e. traditional launchd services) are supported

## CoreAudio

  * Implement AudioFormat and ExtAudioConverter APIs.
  * Implement AUGraph and AudioQueue utility APIs.
  * Implement various Audio Units existing by default on macOS. This includes units providing audio mixing or effects.

## CoreServices

  * Implement LaunchServices APIs for applications and file type mappings, backed by a database.
  * Implement UTI (Uniform Type Identifiers) API, also backed by a database.
