# Darling prefix

A Darling prefix is a [container](internals/basics/containerization.md) overlayed on top of a base macOS-like root file system located in `$installation_prefix/libexec/darling`. The default prefix location is `~/.darling` and this can be controlled with the `DPREFIX` environment variable, very much like `WINEPREFIX` under Wine.

Note that in order to change the prefix location with `DPREFIX`, you should `export` this variable in the current shell before running Darling. Using it only when running Darling (e.g. `DPREFIX=foo darling shell`) will not work as expected.

The container uses overlayfs along with a user mount namespace to provide a different idea of `/` for macOS applications.

When you run an executable inside the prefix for the first time (after boot), `launchd`, the Darwin init process representing the container is started. This init process keeps the root file system mounted.

## Updating the prefix

Unlike Wine, Darling doesn't need to update the prefix whenever the Darling installation is updated. There is one caveat, though: since overlayfs caches the contents of underlying file system(s), you may need to terminate the container to see Darling's updated files:

```
$ darling shutdown
```

Note that this will terminate all processes running in the container.

## Multiple simultaneously running prefixes

Darling supports having multiple prefixes running simultaneously. All `darling` commands will use either the default prefix or the prefix specified by `DPREFIX`, if this environment variable is set. This means, for example, that in order to shutdown a particular prefix, you must set `DPREFIX` to the desired prefix (or unset it, for the default prefix) before running `darling shutdown`.
