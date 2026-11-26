# ObjectVeil 7.1.0-rc.2

This release candidate fixes false missing-file errors when Perforce reads RCS `,v` archives from cloud storage. It includes the single-location storage and throughput changes introduced in the previous candidate.

## RCS archive reads

Cold restores now resolve symlinked depot paths even when the nested local directories have been removed. A previous missing-file result also no longer suppresses a fresh lookup after another process publishes an archive. These fixes apply to managed archives generally, including consolidated RCS text history.

Keep the daemon, tools and preload library from this release together. Stop and fence writers for the upgrade, then restart every managed Perforce process with the new library: replacing the library on disk does not update a running process. Existing namespace identities, configuration, catalog and archive formats are unchanged from 7.1.0-rc.1. Preserve an encrypted deployment's original keystore.

## Set up one cloud location

Configure `primary` with one Azure Blob container, S3 bucket or GCS bucket and a dedicated `path_prefix`. Omit `keystore` and `secondary` for a new plaintext deployment. ObjectVeil stores standalone versions, small-file packs, the catalog and recovery metadata in that location. Generate a non-nil `namespace_id` once and share it between the cache and migration tool. The manifest and job state stay on local disk. [Azure example](https://github.com/rcjackson0926/objectveil/blob/v7.1.0-rc.2/docs/examples/objectveil.azure.example.jsonc) · [Configuration guide](https://github.com/rcjackson0926/objectveil/blob/v7.1.0-rc.2/docs/CONFIGURATION.md)

An existing encrypted namespace still needs its original keystore, namespace ID and prefixes to read its archives. Keep those settings when upgrading. A move to plaintext needs a separate destination and verified migration. [Upgrade guide](https://github.com/rcjackson0926/objectveil/blob/v7.1.0-rc.2/docs/upgrading.md)

## Cache and migration

Packs seal independently and archive writes to different catalog partitions can publish concurrently. A file becomes clean after the recoverable catalog entry is durable. Small files can be restored together from one pack. The migration walker starts work while scanning and can spill large directory listings to disk. [Migration](https://github.com/rcjackson0926/objectveil/blob/v7.1.0-rc.2/docs/migration.md)

Keep `upload_publication_overlap` at its default of `false` for live Perforce submits. A cache draining a backlog already on disk can set it to `true`. Restart the daemon after changing it. [Performance tuning](https://github.com/rcjackson0926/objectveil/blob/v7.1.0-rc.2/docs/TUNING.md)

## Downloads

Run `getconf GNU_LIBC_VERSION` on the x86-64 Linux host. Use the `glibc2.39` tarball on glibc 2.39 or newer. Use `glibc2.17` on glibc 2.17 through 2.38. Each tarball contains all eight executables, the matching preload library and operator documentation. Verify the selected archive with the attached `SHA256SUMS`, then follow `INSTALL.txt` inside it. [Installation instructions](https://github.com/rcjackson0926/objectveil/blob/v7.1.0-rc.2/INSTALL.txt)


The release also provides a Debian/Ubuntu package for glibc 2.39 and a native RHEL 9 x86-64 RPM for glibc 2.34. Their package-manager version is `7.1.0~rc.2`, which sorts below stable `7.1.0`. These are release-candidate packages; follow the coordinated upgrade instructions in `INSTALL.txt`.


## Dependency security update

This build updates `url` to 2.5.9, which fixes the upstream
[RUSTSEC-2026-0592](https://rustsec.org/advisories/RUSTSEC-2026-0592)
denial of service in nested `blob:` URL origin handling. The normal and fuzz
lockfiles both use the patched version.
