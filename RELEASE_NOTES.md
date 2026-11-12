# ObjectVeil 7.0.1

A maintenance release in the same native format generation as 7.0.0. It changes
no configuration, catalog, key or manifest format and needs no conversion.
Replace 7.0.0 by following
[Updating a compatible installation](docs/upgrading.md#updating-a-compatible-installation);
[From 7.0.0 to 7.0.1](docs/upgrading.md#from-700-to-701) lists what behaves
differently afterwards.

## Check `/dev/null` if you ran the 7.0.0 verifier as root

`objectveil-verify` 7.0.0 pointed the Perforce client's settings file at
`/dev/null`. The client rewrites that file by renaming a new one over it, so a
verification run **as root** with Perforce checks enabled replaced the host's
null device with a regular file. On any host where that may have happened,
confirm that `ls -l /dev/null` begins with `c`. A reboot restores the device;
so does `rm -f /dev/null && mknod -m 666 /dev/null c 1 3` as root. 7.0.1 gives
the client a path that cannot exist.

## Fixes

- **Shared-content garbage collection scales.** With deduplication and
  `dedup_gc_single_writer_ack` enabled, the collector that destroys content no
  path references any more now reads the remote catalog and the keystore once
  per pass for a whole batch of records, and does so without pausing uploads or
  cold restores. In 7.0.0 it repeated both reads for every record while uploads
  and restores waited, and it could not finish on a keystore of more than a few
  thousand keys. Records that cannot be collected yet are retried with a delay
  and no longer hold up the others, and every pass logs what it destroyed, what
  it retained and why. See
  [Garbage-collection cost and timing](docs/enabling-content-dedup.md#garbage-collection-cost-and-timing).
- **The manifest database is owner-only.** In the packaged layout the state
  directory is searchable by every local account, which left the list of
  tracked depot paths and content hashes readable. New manifests are created
  `0600`, and the daemon removes world access from an existing one at startup.
- **Configuration mistakes are caught when the file is loaded.** An Azure
  backend without `account_name`, and an S3 backend with only one of
  `access_key` and `secret_key`, are now rejected instead of failing later.
- **Debian package removal no longer stops the service.** It refuses while the
  service or a Perforce server is running, as installation, upgrade and the RPM
  already did. The Debian package now depends on `ca-certificates`.
- **The cross-region warning reaches the EC2 metadata service directly**,
  whatever proxy the environment configures.
- **The example configurations load.** They are now checked by the real parser.
  The packaged example manages the Perforce depots directory, `/p4/1/depots`,
  as the installation guide describes.
- **The guides match the implementation**: credential sources, validation
  rules, defaults, hot-reloadable fields, metrics, the built-in developer
  license and post-expiry behavior. Build-workspace instructions and a socket
  readiness step that no shipped tool can perform were removed.
- `rustix` is updated past RUSTSEC-2026-0486.

## What was tested

[VALIDATION.json](VALIDATION.json) and [RPM_VALIDATION.json](RPM_VALIDATION.json)
record the exact source commit, artifact hashes and results.

- The required Linux gate passed on the release source commit.
- Daemon, preload shim and tools were exercised against a real Perforce 2026.1
  server using the exact packaged executables of both builds.
- Each package was installed, upgraded from the published 7.0.0 package and
  removed in a disposable container, including the refusals while a Perforce
  server or cache daemon is running.
- The RPM build passed the complete test suite on Red Hat UBI 9 user space.

**Not tested for this release:** the live Amazon S3, Azure Blob Storage and
Google Cloud Storage tests, and the S3-backed migration tests, that were
recorded for 7.0.0. Storage behavior in 7.0.1 was exercised only against the
test suite's local backends. No source file of the storage backends, migration,
cold restore, packing or eviction changed since 7.0.0; the garbage-collection
change above does read the keystore and catalog through those backends. The
[migration guide](docs/migration.md) asks for a release whose validation covers
migration and cold reads: for that purpose 7.0.0 remains the validated release.

## Downloads

- **Debian/Ubuntu or generic Linux x86-64** (glibc 2.39 or newer):
  `objectveil_7.0.1_amd64.deb` or `objectveil-7.0.1-linux-x86_64.tar.gz`. Both
  carry byte-identical executables.
- **RHEL 9 x86-64** (glibc 2.34): `objectveil-7.0.1-1.el9.x86_64.rpm`, a separate
  native build. See the [Red Hat guide](docs/redhat.md).

Verify downloads against `SHA256SUMS`. Keep each build's executables and its
preload shim together, and complete the coordinated writer stop in
[INSTALL.txt](INSTALL.txt) before installing, upgrading or removing a package.
Installation never enables or starts the daemon. No license, credentials or
implementation source is bundled.

GitHub's automatic Source code archives contain only this public repository's
documentation and configuration templates.

Uploads remain asynchronous. Losing a writer and its disks before remote
publication can lose recent acknowledged submissions. Retain independent
Perforce metadata backups and a verified archive recovery point; see
[disaster recovery](docs/disaster-recovery.md).
