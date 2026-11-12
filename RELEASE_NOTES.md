# ObjectVeil 7.0.1

A maintenance release in the same native format generation as 7.0.0. It changes
no configuration, catalog, key or manifest format and needs no conversion.
Replace 7.0.0 by following
[Updating a compatible installation](https://github.com/rcjackson0926/objectveil/blob/v7.0.1/docs/upgrading.md#updating-a-compatible-installation);
[From 7.0.0 to 7.0.1](https://github.com/rcjackson0926/objectveil/blob/v7.0.1/docs/upgrading.md#from-700-to-701) lists what behaves
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
  per pass for a whole batch of records. Uploads and cold restores continue
  during those inventories; they may wait during the final key recheck and
  destruction. In 7.0.0 it repeated both reads for every record while uploads
  and restores waited, and it could not finish on a keystore of more than a few
  thousand keys. Records that cannot be collected yet are retried with a delay
  and no longer hold up the others, and every pass logs what it destroyed, what
  it retained and why. See
  [Garbage-collection cost and timing](https://github.com/rcjackson0926/objectveil/blob/v7.0.1/docs/enabling-content-dedup.md#garbage-collection-cost-and-timing).
- **New manifests are owner-only; existing ones lose world access.** In the
  packaged layout the state directory is searchable by every local account,
  which left the list of
  tracked depot paths and content hashes readable. New manifests are created
  `0600`, and the daemon removes world access from an existing one at startup,
  preserving its existing group permissions.
- **GC notices publications already in progress.** Catalog publications report
  their completion or cancellation before releasing the writer drain, including
  publications that started before the collector began watching. Concurrent
  ingestion and deletion also retain their retry wakeups instead of allowing
  an old decision to delay collection for up to a day.
- **Contradictory catalog descriptors are refused.** Each deduplication descriptor
  in a remote delta or snapshot must agree with its enclosing catalog entry on
  the content hash, shared object address and encryption format.
  Invalid records cannot establish authority for restoration, compaction or
  garbage collection, and the remote history is retained for repair.
- **Cloud namespace prefixes are consistent.** Leading and trailing slashes
  are normalized before requests and paginated listings, so a listed object
  can be read again using its returned key. Listings outside the configured
  namespace are rejected.
- **Configuration mistakes are caught when the file is loaded.** An Azure
  backend without `account_name`, and an S3 backend with only one of
  `access_key` and `secret_key`, are now rejected instead of failing later.
- **Debian package removal no longer stops the service.** It refuses while the
  service or a Perforce server is running, as installation, upgrade and the RPM
  already did. The Debian package now depends on `ca-certificates`.
- **The cross-region warning's EC2 metadata query bypasses configured proxies.**
- **The example configurations load.** They are now checked by the real parser.
  The packaged example manages the Perforce depots directory, `/p4/1/depots`,
  as the installation guide describes.
- **The guides match the implementation**: credential sources, validation
  rules, defaults, hot-reloadable fields, metrics, the built-in developer
  license and post-expiry behavior. Build-workspace instructions and a socket
  readiness step that no shipped tool can perform were removed.
- `rustix` is updated past RUSTSEC-2026-0486.

## What was tested

[VALIDATION.json](https://github.com/rcjackson0926/objectveil/releases/download/v7.0.1/VALIDATION.json), [RPM_VALIDATION.json](https://github.com/rcjackson0926/objectveil/releases/download/v7.0.1/RPM_VALIDATION.json)
and [CLOUD_VALIDATION.json](https://github.com/rcjackson0926/objectveil/releases/download/v7.0.1/CLOUD_VALIDATION.json)
record the exact source commit, artifact hashes and results.

The subsequent [Azure validation addendum](https://github.com/rcjackson0926/objectveil/releases/download/v7.0.1/AZURE_VALIDATION.json)
supersedes the Azure-not-tested statements in those original receipts. The
published binaries, release tags and original checksum files are unchanged;
`AZURE_VALIDATION.sha256` verifies the addendum.

- The Linux gate passed on the host runtime source. The final tag adds only
  corrections to test fixtures, test documentation and gate reporting; receipts
  record the exact build and test identities and unchanged production sources.
- Configured workspace line coverage reached 90.48%, using the repository's
  exclusions and including unit-test code. Two 60-second AddressSanitizer fuzz
  targets completed more than 18 million executions without a crash.
- Daemon, preload shim and tools were exercised against a real Perforce 2026.1
  server using the exact packaged executables of both builds.
- Each package was installed, upgraded from the published 7.0.0 package and
  removed in a disposable container, including the refusals while a Perforce
  server or cache daemon is running.
- All 26 native Rust test executables passed on Red Hat UBI 9 user space
  (1,708 harness-reported passes; gated cases are documented in the receipts).

- Live S3, GCS and Azure tests exercise packing, multipart upload, range and concurrent
  cold reads, mutable deduplication, revocation, cold recovery, and corruption
  refusal without partial local publication. The provider fixtures link the
  production engine into their release test executables; reverse migrations
  invoke the sealed migration CLI.
- S3 migration tests invoke the packaged daemon, tools and shim with real
  Perforce: concurrent live imports, busy-file deferral, same-job resume,
  historical archives and shelves, and recovery after actual SIGKILL during
  upload and download. Byte comparisons and namespace checks verify preserved
  source data and remote catalog/key state.
- Test cleanup checks current objects, historical S3 versions, delete markers
  and unfinished multipart uploads. Dedicated GCS test buckets are deleted
  after acceptance; existing storage outside the test namespaces is untouched.

**Azure follow-up validation:** all four Azure cases passed on each published
build: three storage scenarios and an additional backend contract test. The
added case checks conditional key creation without changing existing bytes or
metadata, overwrite/readback, exact ranges,
exclusive listing cursors, idempotent deletion, and the four keystore startup
probes. Authentication used the available account SAS; other Azure credential
modes were not exercised.

The initial container run and a diagnostic repeat refused download resume on
OverlayFS. Captured CLI diagnostics confirmed the existing local-filesystem
safeguard; the same
published executables passed on private ext4 scratch space. No production
guard was weakened.

Azure cleanup verified no active test blobs or uncommitted blocks remained.
Soft-deleted blobs and snapshots, including test key material, remain under the
unchanged seven-day account policy. These tests establish current serving and
revocation behavior, not physical or cryptographic erasure of retained copies.
Production keystores must still
meet the [retention requirements](https://github.com/rcjackson0926/objectveil/blob/v7.0.1/docs/CONFIGURATION.md#azure).

RHEL testing uses UBI 9 user space on the build host's kernel; it does not certify
a booted RHEL kernel, host service activation or enforcing SELinux policy.

## Downloads

- **Debian/Ubuntu or generic Linux x86-64** (glibc 2.39 or newer):
  `objectveil_7.0.1_amd64.deb` or `objectveil-7.0.1-linux-x86_64.tar.gz`. Both
  carry byte-identical executables.
- **RHEL 9 x86-64** (glibc 2.34): `objectveil-7.0.1-1.el9.x86_64.rpm`, a separate
  native build. See the [Red Hat guide](https://github.com/rcjackson0926/objectveil/blob/v7.0.1/docs/redhat.md).

Verify downloads against `SHA256SUMS`. Keep each build's executables and its
preload shim together, and complete the coordinated writer stop in
[INSTALL.txt](https://github.com/rcjackson0926/objectveil/blob/v7.0.1/INSTALL.txt) before installing, upgrading or removing a package.
Installation never enables or starts the daemon. No customer license file,
credentials or implementation source is bundled.

GitHub's automatic Source code archives contain only this public repository's
documentation and configuration templates.

Uploads remain asynchronous. Losing a writer and its disks before remote
publication can lose recent acknowledged submissions. Retain independent
Perforce metadata backups and a verified archive recovery point; see
[disaster recovery](https://github.com/rcjackson0926/objectveil/blob/v7.0.1/docs/disaster-recovery.md).

Dropping a client request does not cancel a write already accepted by an object
store. Ambiguous writes need reconciliation before treating key erasure as
complete; local cancellation alone is not proof of instantaneous cryptographic
erasure.
