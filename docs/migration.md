# Migrating existing Perforce archives with ObjectVeil

Upgrading an existing installation? Follow the
[ObjectVeil upgrade and format guide](upgrading.md) before changing
services, state directories, environment variables or cloud prefixes.


Use a release whose published validation receipt identifies the tested source
and artifact hashes, including migration and cold-read acceptance. This runbook
describes the workflow; it does not itself establish acceptance of a build.

`objectveil-migrate` copies physical archive files directly from a running
Perforce server into packed chunks or standalone objects with a durable catalog.
A new cache can discover these archives using the same primary location and
namespace UUID, independently of the migration machine or its local manifest. The tool preserves the originals
and checks every imported file by downloading it through a separate read-only
cache. Large uploads use the cache's parallel multipart transfer.

For a new plaintext depot, set `primary` to the version-file container or
bucket and prefix. Leave both `secondary` and `keystore` absent.
The same namespace holds the catalog, packs and standalone archives. Fresh
plaintext cloud depots default to the partitioned catalog. Packing
preserves separate paths and checksums even when revisions have identical bytes.
Keep the original configuration and keys to read or resume an encrypted depot.
The local ownership markers and remote storage marker prevent switching a
populated installation between these two formats. Provider-managed encryption at rest can stay enabled.

## Copying a running server

Point `--archive-root` at the existing server's archive directory. Live mode is
the default; `--source-kind live` makes the choice explicit. Perforce can keep
serving reads and accepting submissions. No replica, filesystem snapshot, full
archive-tree duplicate or cache shim in the running server is required for the
copy.

The tool uses Linux kernel read leases to detect files open for writing,
including a writer paused with an open descriptor and writable shared mappings.
It skips busy files and continues with other archives. For a selected file, it
hashes the bytes under a lease, then acquires another lease to capture that
exact file generation into private staging. A changed, replaced or disappeared
file is skipped. The lease is released before encryption, cloud upload and
verification. If a new writer requests access during capture, capture is abandoned
and the writer is allowed to proceed promptly. This can briefly delay a normal
write open; a nonblocking write open may receive `EWOULDBLOCK` and need to retry.
Live copying does not promise zero added writer latency.

Live mode accepts local ext2/3/4, XFS, Btrfs and tmpfs filesystems, and requires
working Linux file leases. Network filesystems, FUSE, overlay filesystems and
unrecognized filesystem types are refused because local leases alone cannot
prove exclusion of their other writers. Run as the archive owner or with
`CAP_LEASE`, plus normal read/traversal permissions. If leases are unavailable,
the tool refuses live capture with an actionable error;
it does not fall back to size, timestamps, process lists or advisory locks.
Check the reported skips after each pass. Repeat the command with the same job
directory to revisit busy files and discover new or changed files.

A pass imports individually captured, completed file generations. It is not a
transactionally consistent snapshot of the whole Perforce server, a continuous
replication service or a deletion mirror. A file may change or be removed after
capture. Later changes and skipped files require another pass or the coordinated
cache writer before cutover. No migration result authorizes deleting the live
archives.

Prepared immutable snapshots and stopped, separately owned replicas remain
supported with `--source-kind snapshot` and `--source-kind replica`. Those modes
require the entire input to stay unchanged and report the sealed generation
actually verified. Their `--source-id` identifies the snapshot or verified replica
generation; a live job instead uses a stable operator-chosen label. Do not change
an existing job's label or source kind to repurpose its state.

For a prepared replica, metadata replication alone does not establish archive
completeness. Verify the required historical archives and shelves before sealing
it. The tool does not provision replicas, replay journals, create snapshots,
change production Perforce configuration or switch the live writer.

## Prepare addressing and source scope

Use the intended writer's validated cache configuration. A plaintext migration
needs one initially empty data namespace. An encrypted migration also needs a
separate key bucket or container with the appropriate retention policy. See
[configuration](CONFIGURATION.md) and [disaster recovery](disaster-recovery.md).

Set `format` to `objectveil/1` and choose a canonical non-nil `namespace_id`.
Preserve that UUID on the migration job and every eventual reader or writer
sharing its remote deployment. Migration creates a native role marker and a
conditional ownership receipt in each configured namespace. Read-only download
requires matching preexisting markers. A conflicting migration
job or foreign objects prevent a new claim. General cache writers do not honor
migration receipts: keep all other writers away from these namespaces until the
import has finished and the writer handoff is coordinated. Never delete an owner
receipt to bypass a conflict.

Set an explicit `primary.path_prefix` and, for encrypted storage, `keystore.path_prefix` for a migration that
runs on a different host or filesystem layout. Otherwise configuration derives
them from the configured depot directory name. Private spool directories never
replace those already resolved cloud prefixes. The source and live cache roots
may have different absolute paths, but each physical archive's relative pathname
must be identical under both roots. Custom or absolute depot Maps need an explicit
layout review; do not assume a logical `//depot/file` is a physical relative path.
Perforce 2026.1 hidden `serverRoot` information is not required.

`--archive-root` must name the complete archive-only tree corresponding to one
configured cache root. Do not select arbitrary P4ROOT contents: database tables,
journals, logs, credentials and configuration are not archive objects. Every
regular file under the selected root is inventoried; unknown names are not silently
discarded. Symlink components in the supplied source root, special files,
inaccessible entries and cache reserved paths fail the import. During live
traversal, a replaced path or a symlink below that root is disclosed as a changed
entry and skipped without following it. Busy and missing entries are also skipped;
sealed modes reject these unsafe entries and source changes. Raw Unix pathname
bytes are preserved.

The upload walker has no fixed file count per directory or per tree. It sorts
raw names in memory while they fit a 16 MiB allowance shared across the active
directories. Wider directories spill to a temporary disk index and are read
back in the same order. A live pass can start copying after it enumerates the
directories leading to the first eligible files. A snapshot pass inventories
the complete source before copying. The index shares the upload job's spool budget.
Upload `--plan` uses temporary space outside the source and leaves the job
directory untouched. A full scratch filesystem or exhausted budget stops the
pass. The traversal depth limit is 128 and relative paths can be up to 4 KiB.

The program verifies the bytes it captures, not that the selected tree contains
every archive referenced by Perforce. Verify source coverage against the matching
Perforce metadata. Include shelves and any additional depot-specific scopes;
edge-local shelves must be checked where they reside. Perforce's
[verification reference](https://help.perforce.com/helix-core/server-apps/cmdref/current/content/CmdRef/p4_verify.html)
documents distinct submitted, shelf, unload and archive verification modes.
Archive-trigger content and depot kinds outside the selected physical tree need
their own explicit migration/verification procedure before whole-server cutover.

## Plan, import and resume

Run on Linux with the same signed license and backend features used by the cache.
The configured `state_dir` holds the installation's licensing anchors and stable
identity. Set it outside the archive source before an initial migration, for
example `/var/lib/objectveil/state`; it must also be separate from the job and
download destination. Upload planning refuses overlapping state and source paths
before creating files. If licensing state already exists, preserve it through an
explicit verified relocation or select an archive-only source that excludes it;
a fresh job or state directory is not a license reset procedure.

Writer, verification and download engines inherit the same authenticated expiry
and installation identity. They use private job audit logs and separate capacity
policies. A short cross-process lock serializes canonical license finalization;
a busy error means retry after the other load finishes. Starting another job does
not start another floating trial window.

Choose an owner-only job directory whose existing parent is outside source,
cache, configuration and local backend trees. The directory is dedicated to this
job; its contents and original source root must remain available for resume. Keep
other processes from modifying, renaming or replacing this private work area while the
command runs; the process lock coordinates migration jobs, not arbitrary programs
running under the same account.

```sh
objectveil-migrate /etc/objectveil/config.json \
  --archive-root /p4/1/depots \
  --source-kind live --source-id existing-server-import \
  --state-dir /var/lib/objectveil-migration/job-001 \
  --jobs 4 --spool-budget-gb 64 --plan
```

Inspect the reported inventory, skips and addressing fingerprint. `--plan`
performs local inspection without uploading. Omit it to import. `--source-kind`
may be omitted for live mode. `--license /secure/license.lic` selects an
explicit license.

For example, if the live cache will manage `/p4/1/depots`, use that archive-only
directory as `--archive-root`. A physical file such as
`/p4/1/depots/project/file.txt,d/1.1` must resolve to the same relative name under
the configured cache root. The argument is not a client workspace or a logical
`//project/...` depot path.

The tool streams copies into a private spool, uses the cache's existing encryption
and publication pipeline, then cold-restores and hashes every archive. It does
not start cache eviction, garbage collection or legacy backfill. Progress is
recorded only after the restored size and BLAKE3 match the captured file.
The verification reader checks remote catalog and key authority before and
after decoding. It hashes the decoded bytes as they stream through the reader,
which avoids writing and syncing a second plaintext file for each verification.
For encrypted standalone uploads, the writer pins the already captured spool
file, verifies its size and digest before minting a key, and authenticates the
plaintext again while encrypting it. This saves one plaintext copy and sync
per standalone archive. Concurrent verifiers checking different keys in the
same catalog shard can share a listing that starts after their requests arrive.

Live import starts processing the first batch while it is still discovering
the rest of the archive tree. Discovery stays bounded by the job's concurrency.
If a later entry stops the scan, batches already published remain in the
owned destination and the job remains incomplete. Fix the reported issue and
resume using the same job. Use upload `--plan` when you need to inspect the
entire source before allowing any writes. Snapshot and replica imports still
verify the complete sealed inventory before publishing anything.

Large stored objects use the same bounded parallel multipart uploader as the
cache, with four in-flight parts of 8 MiB per object. `--jobs` bounds concurrent
archive publishers and defaults to four. It accepts 1 through 64. Each free
publisher can admit another archive while an unrelated upload is still running.
Cold verification has up to 16 workers, also bounded by `--jobs`. Small
archives share packs according to the cache's `pack_max_file_bytes` threshold,
path bucket and compression settings. Standalone transfers can continue while
a partial pack seals. Both modes verify every selected member, including the tail.
The fixed multipart format permits at most 10,000 parts per stored object
(83,886,080,000 bytes, including encryption framing); larger uploads are refused.
The normal configured cache/file limits also apply.

For same-host upload scheduling, add `--upload-first` to the normal upload
command. The uploader refills individual publishing slots as work completes.
The tool accumulates up to eight times `--jobs` archives or the available
spool budget, whichever comes first. It then cold-verifies that window before
uploading again. Verification uses up to 16 workers and respects `--jobs`.
A final partial window also verifies. The ordinary upload mode overlaps
publication and verification across the stream of archives. Both modes keep the captures
and their full staging reservations through verification and record the same
durable per-archive results. Reuse the same job after an interruption.
`--upload-first` applies to uploads only. Compare `last_publication_seconds`
and total verified elapsed time separately on the same source and fresh targets.

For a throughput trial, compare `--jobs 8` and `--jobs 16` on representative
archive sizes if the uploader has the disk space and network capacity.
An active multipart transfer can hold up to 32 MiB of part payloads per
archive, in addition to staging and SDK buffers. Measure job time and both
backend byte counters in the completion log. The same line reports
`backend_gets`, `backend_heads`, `backend_lists` and `backend_puts`.
Compare these counts per verified archive when testing a workload with many
small files. A multipart upload counts as one PUT in this log. The verified
source rate includes cold reads, so it will usually differ from the upload-only
wire rate.

`--spool-budget-gb` reserves space for the bounded active file copies, immutable
upload snapshots, encryption and cold verification. It does not reserve or require
a full copy of the archive tree. Its per-file allowance is deliberately larger
than that file's plaintext size. Ensure the filesystem actually has this free
space plus room for operating-system overhead. Journal and cache database growth share the
workspace allowance with active file work and are refused before extending beyond
it. This accounting is not an operating-system disk quota; a quota or disk error still
causes a resumable failure. Never place this work area on the primary archive
filesystem without independently provisioning its capacity.

Repeat the same command and job directory after a failure or interruption. Source
identity and destination fingerprint must match; credentials may rotate. The
program refuses a replaced source root, foreign ownership, incompatible state
and unverified remote content. Live mode permits file and directory changes
between passes; sealed modes require the original generation to remain unchanged.
Preserve the journal rather than resetting it or starting a new job against the occupied namespace. SIGKILL can leave local staging
files and provider-side incomplete multipart transfers. Local work is reconciled
under the job lock; configure provider lifecycle cleanup for abandoned multipart
uploads because machine death cannot execute an abort request.

Back up the dedicated job state. For a sealed source, preserve its matching
generation too. If the entire migration machine and its journal are lost, retain the partially populated cloud
namespaces for diagnosis; a fresh job cannot claim them as empty or impersonate
their owner. The original primary archives remain the recovery source. Restart
in newly dedicated namespaces rather than deleting ownership records or guessing
which partial objects are complete.

`--verify-only` repeats cold verification of the eligible captured generations
in the current recorded pass, provided that pass's inventory scan finished. If a
newer scan was interrupted, it refuses verification rather than falling back to
an older completed pass; run a normal import pass first. In live mode it does not
scan the current source again or upload later changes, and it retains the
recorded skips. Run a normal import pass to discover and publish new generations.

Live reports use `live_pass_complete` when the pass has no skips and
`live_pass_partial` otherwise. `files_skipped` counts deferred source entries,
including a vanished directory when its individual files cannot be enumerated;
it is not necessarily an exact count of missing archives. The live summary records
busy, changed and missing entries, with a bounded sample of paths. Exit status 0 means
the plan/pass completed without skips, 2 means a live pass completed with skipped
files, and 1 means an error or refusal. Inspect the status and scope in the JSON
output and owner-only `report.json`; even a complete live pass verifies individual
captures rather than the server's current state. An earlier successful receipt
does not supersede a failed later attempt.

## Verify and hand off

Preserve the matching Perforce metadata/checkpoints/journals separately. Before
cutover, test an isolated metadata clone with empty local archives and the actual
cache/shim against only the target cloud. Disable upstream replica/on-demand
fallback, which could hide a missing cloud object. Check historical revisions,
RCS archives, shelves and installation-specific depot scopes; inspect both output
streams and exit status for missing or damaged content.

Finish the migration process before starting another writer to its namespaces.
The writer's remote catalog remains the source of discovery for fresh caches.
Its private `writer` state directory also retains the imported manifest receipts;
preserve it if planning a validated state handoff. Do not overwrite a running or
previously populated daemon state directory with these files. Take a backup and
validate the complete state/mapping before any state transfer.

The live writer must use the reviewed shim and externally fenced single-writer
protocol. Reconcile skipped files and changes after capture before enabling source
space reclamation. A live import has no single server-wide generation watermark.
Deploying or failing over shim-bearing processes is a separate coordinated operational step; importing archives does not modify running
p4d processes. The tool never deletes original source archives or authorizes
their removal. Normal cache eviction may reclaim proved-durable files after the
coordinated writer is operating correctly.

The cache remains asynchronous after handoff. Complete loss of the only writer
disk before remote payload, key and catalog publication can still lose recent
writes. Migration does not establish a zero-loss recovery point for that window.

## Copy archives back from cloud storage

Use `--download` with the configuration addressing the original data namespace
and its matching separate key namespace:

```sh
objectveil-migrate /etc/objectveil/config.json --download \
  --archive-root /restore/depots \
  --state-dir /var/lib/objectveil-migration/download-001 \
  --jobs 4 --spool-budget-gb 64
```

`--archive-root` is now the local output directory. Its parent must exist; the
output directory and its descendant directories must be private and owned by the
account running the tool. Existing-file resume requires Linux kernel read leases;
a file currently open for writing is refused rather than trusted.
Restore outside the archive tree of a running Perforce server. Use a dedicated,
initially empty directory for an exact copy of the selected generation. Completion verifies every selected path. An existing destination
can also contain unrelated files, which the tool leaves in place.
The tool preserves relative archive names, including non-UTF-8 Unix names, and
reconstructs ordinary plaintext version files from standalone, deduplicated and
packed cloud objects. Supply `--license` if needed. Download mode does not use
`--source-id`, `--source-kind` or `--verify-only`.

With a partitioned catalog you can separate the inventory from the payload
restore. Stop catalog writers, compaction, and garbage collection for the
inventory. Run the download with `--plan --sealed-inventory` using a new job
directory. Wait for `download_planned` and a `sealed_catalog_fingerprint` in
the report before resuming writers. Then run the same command with
`--sealed-inventory` and omit `--plan`. This restore and subsequent retries use
the exact archive selection saved on local disk. Unrelated submissions after
the seal do not trigger a new inventory. Every selected path must still have
the recorded descriptor and usable keys at verification time. If a selected
archive was superseded or obliterated, the restore reports the conflict. A
new job directory can capture a later selection. Pack compaction can also
relocate an unchanged member. Keep payload repacking paused for the selected archives until restore
finishes, or create a new sealed plan after a relocation.

The partition inventory reads a bounded page of namespace keys at a time and
checks the selected catalog metadata before and after the walk. A changed
generation interrupts the live scan. A retry can reuse completed partitions
when all catalog validators were strong and still match. For metadata without
a strong ETag, the reader downloads the bounded object and fingerprints its
bytes on each pass. Allow local workspace for the on-disk inventory of the
whole depot plus temporary verification files. Reopening the job streams the
restore tree once to account for interrupted staging files, including in
directories with more than 100,000 children. Long-term recovery from a sealed
inventory also requires preserving its matching payloads and any required encryption keys.

When restoring plaintext packs, ObjectVeil groups selected members by pack,
downloads each selected pack once, validates its address and footer, and checks
each selected member's length and digest before publishing its local file.
The default download rechecks the current catalog as each group is processed.
For a partitioned catalog whose writers, obliterates, catalog maintenance and
pack compaction are fenced for the entire download, add `--quiescent-catalog`
to avoid repeated metadata reads.
It checks the catalog generation again before reporting success. If writers
must continue during a download, omit this flag. A previously sealed inventory
alone does not establish that the current catalog is quiescent.

A default download reads a complete current catalog inventory before publishing files.
It authenticates/decrypts each selected file, checks its size and BLAKE3, then
publishes it atomically without replacing an existing path. Existing files must
match exactly and are checked again on every invocation; differing contents,
symlinks and unsafe paths stop the job without overwriting them. A conflicting
file requires an operator decision outside the tool. No cloud objects, catalog
records, keys or original archives are deleted or rewritten.

Repeat the same command/state directory after interruption. The new invocation
reads current remote authority again and verifies existing output instead of
trusting a previous completion receipt. Changes observed by the before/after catalog and key checks fail closed; retry
after the change settles. A captured generation may be superseded after those
checks, including during local publication. A completed download
is a finite set of individually verified archives, not a transactionally consistent
whole-server snapshot. Keep the matching Perforce databases/checkpoints/journals
separately and validate their archive coverage before bringing a restored server
online.

Download `--plan` inventories remote metadata and writes private local job state,
but downloads no archive payloads. It does not have upload mode's local-only
planning semantics. Current catalog objects are authoritative: the tool refuses
legacy raw-path or route-only namespaces rather than silently omitting their
archives or resurrecting an obsolete payload. Download requires a healthy, complete catalog. `remote_catalog_complete` means
that its listing and reads finished; it does not independently prove that every
payload or Perforce metadata reference has a catalog entry. Missing catalog
history needs the separate recovery procedure before downloading. Malformed,
capped or unreadable catalog data stops the download. The legacy layout limits an inventory to one million namespace objects and one
million archives. Partitioned downloads process a bounded page at a time and
checkpoint the completed partitions in the private job ledger. A changed
catalog invalidates an incomplete pass before any such inventory is accepted.

Packed member decoding also enforces the cache's configured chunk size ceiling.
Use pack size settings compatible with the writer that created those chunks;
restoring older, larger chunks with smaller limits fails safely instead of
allocating from untrusted lengths.

The spool allowance bounds private job metadata and in-flight temporary data,
not the final restored archive tree. Provision enough destination space for the
full result plus temporary copies. Interrupted temporary files remain in the publisher's reserved staging directories and are
counted conservatively at startup; the tool never sweeps arbitrary destination
contents. Startup inspects the existing restore tree once; it does not rescan
the full result after each batch.
