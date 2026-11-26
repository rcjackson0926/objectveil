# ObjectVeil 7.1.0

This Linux release contains the cache daemon, its matching Perforce preload
shim, the migration uploader/downloader, and the verification and recovery
utilities and native initializer. This release requires `objectveil/1` configuration,
a deployment namespace UUID and native stored formats. Customer license files
use the version 2 signed format; no customer license file or private configuration
is bundled. Without a license file, the binary's developer fallback permits
1 TiB for 30 days from its build date. Production deployments must override it;
see [Licensing](docs/LICENSING.md).

## Download the release

Run `getconf GNU_LIBC_VERSION` on your x86-64 Linux server. Choose the
`glibc2.39` archive on glibc 2.39 or newer, or `glibc2.17` on glibc
2.17 through 2.38. Verify the download against the release’s `SHA256SUMS`
and keep the daemon and preload library from the same archive together.
A Debian/Ubuntu package (glibc 2.39) and native RHEL 9 RPM (glibc 2.34)
are also available. Their package version is `7.1.0`; see `INSTALL.txt`.
[Release downloads](https://github.com/rcjackson0926/objectveil/releases/tag/v7.1.0) · [Installation](INSTALL.txt)

This page and its configuration examples describe the 7.1.0 release.
For an installed 7.0.1 server, use the [7.0.1 documentation](https://github.com/rcjackson0926/objectveil/tree/v7.0.1/docs).

GitHub also displays automatic **Source code (zip)** and **Source code (tar.gz)** links. Those archives contain only this public repository’s documentation and configuration templates. The Rust implementation is maintained privately and is not included in those archives. Download the named Linux release assets for the executable tools.

## Included tools

| File | Purpose |
| --- | --- |
| `bin/objectveil` | Cache daemon |
| `bin/objectveil-initialize` | Explicit local native archive-root initialization |
| `bin/libobjectveil_shim.so` | Matching Perforce preload shim |
| `bin/objectveil-migrate` | Resumable live archive upload and cloud download |
| `bin/objectveil-verify` | Archive and checkpoint/journal verification |
| `bin/objectveil-manifest-summary` | Offline manifest summary preparation |
| `bin/objectveil-rebuild-catalog` | Fenced catalog reconstruction |
| `bin/objectveil-rescan` | Explicit cache reconciliation |
| `bin/objectveil-dedup-measure` | Deduplication measurement |

The uploader detects active writers, reports skipped files, and uses the
cache's normal packing and parallel multipart uploads. For a new cloud
deployment, configure one `primary` bucket or container for Perforce version
files. Small files, standalone revisions and the catalog share that location.
Omit `keystore` and `secondary` for this plaintext layout. Existing encrypted
deployments require their original keystore. `--download` reconstructs the supported current cloud
catalog into a separate local tree and preserves conflicting local files.

## Documentation

- [Red Hat Enterprise Linux RPM installation](docs/redhat.md)

- [Installation and upgrades](INSTALL.txt) and [existing-installation upgrade](docs/upgrading.md)
- [Configuration reference](docs/CONFIGURATION.md) and [examples](docs/examples/)
- [Operations](docs/OPERATIONS.md) and [quick start](docs/QUICKSTART.md)
- [Live upload, download and resume](docs/migration.md)
- [Disaster recovery and durability](docs/disaster-recovery.md)
- [Chunk packing](docs/enabling-chunk-packing.md) and [deduplication](docs/enabling-content-dedup.md)
- [Tuning](docs/TUNING.md) and [architecture](docs/ARCHITECTURE.md)
- [Licensing](docs/LICENSING.md) and [evaluation terms](EVALUATION-TERMS.md)

In each tarball, `BUILD-INFO.json` identifies the release and source revision and `ABI.json` lists the binary dependencies. `SHA256SUMS` verifies
the files inside this bundle. The release-level checksum file verifies the
download assets. The initializer, daemon and migration executable carry
verified integrity seals.

Writer upgrades require a coordinated stop of p4d, ObjectVeil and any previous
cache daemon, followed by restarting every p4d with the matching new shim.
Incompatible persisted formats require an explicit verified offline conversion;
installation does not adopt or reset them. Run the initializer only after following
the archive-root preparation instructions in INSTALL.txt. Archive uploads remain
asynchronous, so independent Perforce metadata backups and a defined recovery
point are required. See the installation and recovery guides before cutover.

Request an evaluation license at [ObjectVeil](https://rcjacksonconsulting.com/objectveil-product.html), or use
a valid version 2 ObjectVeil license. Production licensing and support:
rusty@rcjacksonconsulting.com.
