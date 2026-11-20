# ObjectVeil 7.1.0-rc.1

This release candidate adds single-location cloud storage for Perforce version files and improves throughput for parallel submits, migration and cache catch-up.

## Set up one cloud location

Configure `primary` with one Azure Blob container, S3 bucket or GCS bucket and a dedicated `path_prefix`. Omit `keystore` and `secondary` for a new plaintext deployment. ObjectVeil stores standalone versions, small-file packs, the catalog and recovery metadata in that location. Generate a non-nil `namespace_id` once and share it between the cache and migration tool. The manifest and job state stay on local disk. [Azure example](https://github.com/rcjackson0926/objectveil/blob/v7.1.0-rc.1/docs/examples/objectveil.azure.example.jsonc) · [Configuration guide](https://github.com/rcjackson0926/objectveil/blob/v7.1.0-rc.1/docs/CONFIGURATION.md)

An existing encrypted namespace still needs its original keystore, namespace ID and prefixes to read its archives. Keep those settings when upgrading. A move to plaintext needs a separate destination and verified migration. [Upgrade guide](https://github.com/rcjackson0926/objectveil/blob/v7.1.0-rc.1/docs/upgrading.md)

## Cache and migration

Packs seal independently and archive writes to different catalog partitions can publish concurrently. A file becomes clean after the recoverable catalog entry is durable. Small files can be restored together from one pack. The migration walker starts work while scanning and can spill large directory listings to disk. [Migration](https://github.com/rcjackson0926/objectveil/blob/v7.1.0-rc.1/docs/migration.md)

Keep `upload_publication_overlap` at its default of `false` for live Perforce submits. A cache draining a backlog already on disk can set it to `true`. Restart the daemon after changing it. [Performance tuning](https://github.com/rcjackson0926/objectveil/blob/v7.1.0-rc.1/docs/TUNING.md)

## Downloads

Run `getconf GNU_LIBC_VERSION` on the x86-64 Linux host. Use the `glibc2.39` tarball on glibc 2.39 or newer. Use `glibc2.17` on glibc 2.17 through 2.38. Each tarball contains all eight executables, the matching preload library and operator documentation. Verify the selected archive with the attached `SHA256SUMS`, then follow `INSTALL.txt` inside it. [Installation instructions](https://github.com/rcjackson0926/objectveil/blob/v7.1.0-rc.1/INSTALL.txt)
