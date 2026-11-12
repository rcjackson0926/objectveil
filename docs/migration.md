# Migrating existing Perforce archives with ObjectVeil

Upgrading an existing installation? Follow the
[ObjectVeil upgrade and format guide](upgrading.md) before changing
services, state directories, environment variables or cloud prefixes.


Use a release whose published validation receipt identifies the tested source
and artifact hashes, including migration and cold-read acceptance. This runbook
describes the workflow; it does not itself establish acceptance of a build.

`objectveil-migrate` copies physical archive files directly from a running
Perforce server into the cache's normal packed chunks or standalone objects,
separate key storage and causal catalog. A new cache can discover these archives without
the migration machine or its local manifest. The tool preserves the originals
and checks every imported file by downloading it through a separate read-only
cache. Large uploads use the cache's parallel multipart transfer.

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

Use the intended writer's validated cache configuration. Cloud data and keys
must use different buckets/containers, with independently verified key-retention
policies. See [configuration](CONFIGURATION.md) and
[disaster recovery](disaster-recovery.md). Existing Azure account soft deletion
or GCS bucket retention can defeat permanent key erasure; a successful archive
copy does not validate erasure policies.

Set `format` to `objectveil/1` and choose a canonical non-nil `namespace_id`.
Preserve that UUID on the migration job and every eventual reader/writer sharing
its remote deployment. The migration creates native role markers in empty data
and key namespaces before claiming its own job receipts. Read-only download
requires matching preexisting role markers; it never creates remote identities.

Use a dedicated, initially empty effective data namespace and key namespace.
The program claims both using conditional ownership receipts. Another migration
job or foreign objects prevent a new claim. General cache writers do not honor
migration receipts: keep all other writers away from these namespaces until the
import has finished and the writer handoff is coordinated. Never delete an owner
receipt to bypass a conflict.

Set explicit `primary.path_prefix` and `keystore.path_prefix` for a migration that
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
Traversal accepts at most 100,000 entries per directory, 16 MiB of buffered
pathname names across the walk, directory depth 128 and relative paths of
4 KiB. Exceeding a bound is an error, not a busy-file skip.

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

Large stored objects use the same bounded parallel multipart uploader as the
cache: four in-flight parts of 8 MiB per object. `--jobs` bounds concurrent archive
work and defaults to four; it accepts 1 through 64. Small archives follow the same `pack_max_file_bytes` threshold, path bucket,
compression choice and per-file encryption as the cache. Eligible files share a
chunk; larger files follow the ordinary standalone/dedup route. Packing therefore
does not bypass per-path key revocation. The finite importer flushes its bounded
worker batch before cold-verifying every member, including a final partial chunk.
Batch boundaries can produce different chunk membership than a long-running daemon;
the storage format, addressing and eligibility rules are the same. Encryption uses
fresh keys/nonces, so objects are not promised to be byte-for-byte identical to a
separate hypothetical run of the daemon. The fixed multipart format permits at most 10,000 parts per stored object
(83,886,080,000 bytes, including encryption framing); larger uploads are refused.
The normal configured cache/file limits also apply.

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
Use a separate restore tree, not the archive tree of a running Perforce server.
The tool preserves relative archive names, including non-UTF-8 Unix names, and
reconstructs ordinary plaintext version files from standalone, deduplicated and
packed cloud objects. Supply `--license` if needed. Download mode does not use
`--source-id`, `--source-kind` or `--verify-only`.

The tool reads a complete current catalog inventory before publishing files.
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
capped or unreadable catalog data stops the download. The current inventory
limits are one million namespace objects and one million archives, plus the
normal per-shard authority limits; exceeding a limit is an error.

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
