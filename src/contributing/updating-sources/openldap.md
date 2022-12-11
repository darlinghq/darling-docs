# Additional Update Guidelines for OpenLDAP

As a reminder, these are *guidelines*. The version you update to might make some of these steps obsolete, so make sure you understand what they do and why they're necessary.

## Notes

I replaced `--enable-bdb  --x-libraries=${SDKROOT}/usr/local/BerkeleyDB/lib` with `--disable-slapd`, since I was having trouble building the BerkeleyDB libraries.

## Generate Headers

Keep in mind that the following steps should be executed on either macOS or Darling.

```sh
# Inside the OpenLDAP source
cd OpenLDAP
./configure --disable-shared --disable-cleartext --disable-slapd --enable-aci=yes --enable-overlays=yes --enable-dynid=yes --enable-auditlog=yes --enable-unique=yes --enable-odlocales=yes --enable-odusers=yes

# `make depend` will also generate symbolic-links to some source files. Don't upload those files.
make depend
```

If you are generating the headers from Darling, you might need to manually copy over `ldap_rb_stats.h` from the previous version. At the time of writing, Darling does not include the `dtrace` program (which is needed for `make` to generate the file).

## References

* `[OpenLDAP Source]/OpenLDAP/INSTALL`
* `[OpenLDAP Source]/Makefile` - Where I got the configuration flags from.
