# Additional Update Guidelines for BerkeleyDB

As a reminder, these are *guidelines*. The version you update to might make some of these steps obsolete, so make sure you understand what they do and why they're necessary.

## Disclaimer

I have not been able to successfully generate working headers. I always seem to run into build issues when I try to use the newly generated headers.

For the time being, I just use the generated headers from the previous version, at least they seem to work (for now).

## Generate Build Unix Header

The only additional step you need to do (before BerkeleyDB can build) is to generate the headers in `build_unix`.

Keep in mind that the following steps should be executed on either macOS or Darling.

```sh
# Inside the BerkleyDB source
cd db/build_unix
../dist/configure --disable-java --disable-shared --prefix=/usr docdir=/usr/BerkeleyDB/docs
```

## References

* `[BerkeleyDB Source]/db/README`
  * `[BerkeleyDB Source]/db/docs/index.html`
  * `[BerkeleyDB Source]/db/docs/ref/build_unix/intro.html`