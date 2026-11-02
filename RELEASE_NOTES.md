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
documentation and configuration templates. Download the named Linux tar, Debian
or Red Hat RPM assets for the executable tools.

The packaged operating guides specify native catalog version 3, conservative
retention of reusable storage addresses, shared-content deletion limits, explicit
writer cutover, absolute restore deadlines and best-effort access analytics. The
cost model distinguishes payload requests from full authority and provider costs.
The original Debian and generic-tar runtime files remain identical to the artifacts used for the recorded cloud,
Perforce, licensing and recovery tests.

## Red Hat Enterprise Linux 9 RPM

The separate native `objectveil-7.0.0-1.el9.x86_64.rpm` is built for RHEL 9
with glibc 2.34. It includes the same eight tools and matching preload library,
plus the systemd unit, examples and guides. Keep its runtime components together;
the original Debian and generic-tar downloads require glibc 2.39.

Read the [Red Hat guide](https://github.com/rcjackson0926/objectveil/blob/main/docs/redhat.md)
and complete the coordinated writer stop before package installation or removal.
For online migration without replacing a mapped library, the guide explains
how to extract the RPM separately and run only the migration command.
Installation never enables/starts the daemon or initializes operator state.

[The RPM validation report](https://github.com/rcjackson0926/objectveil/releases/download/v7.0.0/RPM_VALIDATION.json)
records the native suite, exact packaged Perforce integration, 69 Python controls,
and 174 installation/upgrade/removal assertions. Testing used official Red Hat
UBI 9 user space on an Ubuntu host; validate your actual RHEL kernel, service
and SELinux policy before enabling eviction. No new cloud-provider test or
Red Hat certification is claimed for this packaging addition. The original
Debian and tar runtime bytes are unchanged.
