# Darling prefix

Darling prefix is a [container](internals/basics/containerization.md) overlayed on top of a base macOS-like root file system located in `$installation_prefix/libexec/darling`. The default location is `~/.darling` and this can be controlled with the `DPREFIX` environment variable, very much like `WINEPREFIX` under Wine.

The container uses overlayfs along with a user mount namespace to provide a different idea of `/` for macOS applications.

When you run an executable inside the prefix for the first time (after boot), `launchd`, the Darwin init process representing the container is started. This init process keeps the root file system mounted.

Note: Do not put an ending `/` in the variable - Darling uses `${DPREFIX}.workdir` directly as overlayfs's working directory,
and `someplace/.workdir` breaks it. Also, do not create it beforehand (unlike WINE) - if the directory exists and empty, it confuses darling.

## Updating the prefix

Unlike Wine, Darling doesn't need to update the prefix whenever the Darling installation is updated. There is one caveat, though: since overlayfs caches the contents of underlying file system(s), you may need to terminate the container to see Darling's updated files:

```
$ darling shutdown
```

Note that this will terminate all processes running in the container.

