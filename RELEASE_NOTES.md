# ObjectVeil 7.0.0

This release generation requires native `objectveil/1` configuration, a non-nil
deployment namespace UUID, matching storage role markers, and a vendor-signed
version 2 license. It does not automatically adopt incompatible persisted data.
Read [the upgrade guide](docs/upgrading.md) before replacing an installation.

The artifact set contains eight executable tools and one matching preload
library. `objectveil-initialize` is the ninth runtime artifact and establishes
explicit local archive-root ownership before daemon startup. Its default is an
empty archive root; `--existing-archives` requires a reviewed archive-only tree
and does not create a consistent Perforce snapshot or convert control state.

Migration uses the cache's publication and cold-read paths, including packing,
encryption, deduplication and parallel multipart transfer. Live upload preserves
source archives and reports skipped busy or changed files. Download restores a
supported current native catalog into a separate owned tree and preserves
conflicting files. Neither operation supplies Perforce metadata backups or an
automatic writer cutover.

Recovery and resource safeguards include complete catalog authority before cold
restoration or eviction, retained payloads when safe deletion cannot be proved,
bounded packing/recovery work, and preservation of installation license, clock
and trial identity across migration jobs. A separate job directory must not be
used to reset an installation's entitlement state.

Uploads remain asynchronous. Losing a writer and its disks before remote
publication can lose recent acknowledged submissions. Retain independent
Perforce metadata backups and a verified archive recovery point; see
[disaster recovery](docs/disaster-recovery.md).

[VALIDATION.json](VALIDATION.json) records the exact source and artifact hashes,
Linux gates, cloud-provider functional tests, packaged migration and cold reads,
permissions and crash-recovery checks, and synthetic licensing tests. Test-resource
cleanup is reported separately, including recoverable Azure history retained by
the account's seven-day soft-delete policy. No license, credentials, implementation
source or private review records are bundled.

GitHub's automatic Source code archives contain only this public repository's
documentation and configuration templates. Download the named Linux tar or Debian
assets for the executable tools.
