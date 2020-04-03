# Containerization

Darling supports use of multiple prefixes (virtual root directories), very much like Wine. Unlike Wine, Darling makes use of Linux's support for various user-controlled namespaces. This makes Darling's prefixes behave a lot more like Docker/LXC containers.

## Implementation

The implementation fully resides in the ''darling'' [binary](https///github.com/darlinghq/darling/blob/master/src/startup/darling.c), which performs several tasks:


*  Create a new mount namespace. Mounts created inside the namespace are automatically destroyed when the container is stopped.

*  Set up an overlayfs mount, which overlays Darling's readonly root tree (which is installed e.g. in ''/usr/local/libexec/darling'') with the prefix's path. This means the prefix gets updated prefix contents for free (unlike in Wine), but the user can still manipulate prefix contents.

*  Use ''pivot_root()'' to change the root directory. The original root is accessible via ''/Volumes/SystemRoot'' (from inside the container).

*  Set up a new PID namespace. A virtual "init" process is started, which reaps zombie processes. The init process is also used for joining the namespace. (In future, launchd should be used here.)

More namespaces (e.g. UID or network) will be considered in future.

## Caveats


*  When you make changes to Darling's installation directory (e.g. ''/usr/local/libexec/darling''), you must stop running containers (via ''darling shutdown'') so that the changes take effect.

