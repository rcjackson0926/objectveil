# ObjectVeil 7.0.1

This repository contains public documentation and configuration templates.
Executable tools are supplied as named Linux release assets. Start with
[INSTALL.txt](INSTALL.txt); the `bin/`, `MANIFEST.txt` and inner `SHA256SUMS`
paths below refer to the extracted tar bundle.

This Linux release contains the cache daemon, its matching Perforce preload
shim, the migration uploader/downloader, and the verification and recovery
utilities and native initializer. This release requires `objectveil/1` configuration,
a deployment namespace UUID, native stored formats, and version 2 signed licenses.
No license or private configuration is bundled.

Start with [INSTALL.txt](INSTALL.txt). Choose the build for your system:

- **RHEL 9 x86-64:** `objectveil-7.0.1-1.el9.x86_64.rpm`, built against glibc 2.34. See [Red Hat installation](docs/redhat.md).
- **Debian/Ubuntu or generic Linux x86-64:** the `.deb` or named Linux tar bundle, requiring glibc 2.39 or newer.

Keep each build's executables and matching shim together. The RPM is a separate native build; [RPM_VALIDATION.json](RPM_VALIDATION.json) records its tests, hashes and container-testing limits.

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
cache's normal packing, compression, encryption, deduplication and parallel
multipart pipeline. `--download` reconstructs the supported current cloud
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

`MANIFEST.txt` identifies the release and source revision. `SHA256SUMS` verifies
the files inside this bundle. The release-level checksum file verifies the
download assets. The release and Rust workspace versions identify this native
format generation. [VALIDATION.json](VALIDATION.json) records the tested source,
artifact hashes, the checks run for this release and the checks that were not.
The initializer, daemon and migration executable carry verified integrity seals.

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
