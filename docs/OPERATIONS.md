# Operations

Upgrading an existing installation? Follow the
[ObjectVeil upgrade and format guide](upgrading.md) before changing
services, state directories, environment variables or cloud prefixes.


## Installation

Start with [INSTALL.txt](../INSTALL.txt). Verify the selected Linux release
assets before installing; this public repository does not include implementation
source or a build toolchain. Use the complete matching artifact set:

- `objectveil` — daemon
- `objectveil-initialize` — native local archive-root initialization
- `objectveil-migrate` — upload/download migration
- `objectveil-verify` — archive verification
- `objectveil-manifest-summary` — offline manifest summary
- `objectveil-rebuild-catalog` — fenced catalog reconstruction
- `objectveil-rescan` — reconciliation
- `objectveil-dedup-measure` — deduplication measurement
- `libobjectveil_shim.so` — matching preload library

Executable examples below use the Debian paths. For a tar installation, use
absolute paths to the same tools inside the verified versioned bundle.

### Cloud backends

The cloud backends (S3, Azure, GCS) are a single, always-compiled
`object_store`-backed wrapper — there are no per-cloud feature gates or
per-cloud build variants, and one binary serves every backend. NFS is
also always compiled (a custom backend with no optional dependencies),
so the default feature set is empty. Which backend a given deployment
uses is chosen entirely by the `primary`/`secondary` config, not at
build time.

## Debian Package

Install the verified `objectveil_7.0.0_amd64.deb` asset as described in
[INSTALL.txt](../INSTALL.txt). Complete the coordinated writer stop first; the
package does not stop or restart Perforce for you.

Package contents include all eight executables in `/usr/bin`, the matching
`/usr/lib/objectveil/libobjectveil_shim.so`, the systemd unit, the example
configuration and operator documentation. No signed license is included.

The packaged `systemd` unit expects:

- `/etc/objectveil/config.json` to exist
- `daemon = false` in that config so `systemd` owns process supervision
- the `perforce` user and group to exist

The packaged unit runs as `perforce` and lets `systemd` create:

- `/run/objectveil`
- `/var/log/objectveil`
- `/var/lib/objectveil`

The packaged example config is aligned with that layout.
The packaged unit also sets `TimeoutStartSec=0` so large depots are not killed
by `systemd` while the daemon is walking existing files.

The supplied tools must be kept together when installing, upgrading or rolling
back. Preserve native state and follow [upgrading](upgrading.md) for format
compatibility requirements.

## Deployment Model

`objectveil` has two distinct runtime modes:

- Write mode: watches local depot writes with fanotify, uploads dirty files to the primary backend, serves cold restores, and evicts safely uploaded files. All archive writers must use the coordinated-write shim.
- Read-only mode: skips fanotify and uploads. New archive opens check remote catalog/key authority, including warm local hits. Managed archive writes and deletions are rejected, including explicit daemon deletion requests.

Read-only mode is for readers of an already populated backend, not a substitute
for write mode on a submit server. It requires remote authority to be available
even for warm reads. Deploy the daemon and shim together and restart p4d to load
the shim. Changing modes requires restarting p4d too because the shim caches its
verified role for the process lifetime. See [disaster recovery](disaster-recovery.md)
for the supported writer paths, upgrade fence and asynchronous durability limit.

Eviction retains objects whose stored size is unknown or exceeds the configured
cold-restore limits. It continues to consider other eligible files. For retained
oversized objects, increase the cache/restore budget and restart the daemon;
live watermark changes do not change backend limits fixed at startup. Allow
space for pending uploads and restore staging as well as the steady cache target.

Write mode needs Linux fanotify privileges:

```bash
sudo setcap cap_sys_admin+ep /usr/bin/objectveil
```

Reader-only deployments can use `read_only = true` without this capability.

### Socket access across Linux users

Run the serving daemon and application under the same Linux UID. The packaged
Perforce deployment uses `perforce` for both and grants the daemon its required
fanotify capability. Restored files are private and owned by the daemon; a
shared socket group or an entry in `shim_allowed_uids` does not make a
cross-UID archive-serving deployment work. Do not run a root daemon for an
unprivileged application and assume its restored files become application-owned.

A freshly created default `.objectveil` state directory has mode `0700`, and
its ownership marker is `0600`. Existing selected state permissions and
operator-precreated directory modes/groups are preserved. For any separately
qualified protocol/monitoring clients, every ancestor directory must permit
search, and the socket mode/group must permit connection. An operator can
precreate daemon-owned state with a trusted group and mode `0750` or `0710`
when that access is needed. Keep socket modes restricted (normally `0660`),
and do not grant group/world write on state. UID allowlists do not select a
group or grant filesystem permissions.

The shim also accepts a daemon peer only when it runs as the application's UID
or as root. This authentication rule does not change the restored file's owner
or mode. Add every intended connecting UID to `shim_allowed_uids` and
`access_allowed_uids`, and every archive writer to `watcher_allowed_uids`, but
keep serving processes under the matching UID for this release.

Verify readiness and an actual cold read through the matching shim **as the
application's real UID**, including after a daemon restart. Use a controlled,
independently verified test archive; do not remove real archives just to
manufacture a cold read. For existing state, preserve its contents and ownership
while planning any permission change. Do not recursively broaden state,
restored-file or license permissions to solve an access failure.

### Perforce 2026.1 hidden server information

Keep the default `dm.info.hide=1`. The daemon and shim do not discover depot
paths from `p4 info`; they use the configured depot path. Verification needs
the server's `Case Handling` value, which remains available with the root and
address hidden. A missing `Server root` field does not require a configuration
change or broader disclosure of server information.

## Verification jobs and restarts

One verifier job owns its state directory for the full scan, integrity pass,
and log flush. `--estimate` uses the same ownership lock and cannot run against
an active job's state/log namespace. Use separate dedicated state directories
and log paths for independent jobs.

Cancellation does not stop filesystem operations that have already started.
Those writes retain ownership until they finish, including queued log writes
that could rotate the log. Never unlink `.lock` to bypass a busy job: doing so
can allow two owners to mutate the same state. Investigate the owning process
and preserve its state for resume.

Estimation can create ownership-control files and a progress log, but it does
not reset or clean saved scan progress, including when `--reset-state` is also
supplied. It still reads provider metadata; see [verification cost tiers](verify-cost-tiers.md).
Rejected concurrent runs report errors on stderr and leave the active job's
log unchanged. Panics also use stderr; normal owned-run diagnostics go to the
progress log.

## Quick Start

See [`QUICKSTART.md`](QUICKSTART.md) for the full narrative walkthrough
(storage provisioning, read-only-first rollout, wiring up `p4d`,
verification). Terse version:

1. Install and verify the complete matching release artifacts.
2. Create a config file from one of the examples in [`docs/examples`](examples).
3. Set `format` to `objectveil/1`, generate the deployment namespace UUID,
   and run `objectveil-initialize /etc/objectveil/config.json` as the archive
   owner. An existing archive-only root requires a reviewed `--existing-archives`
   attestation; incompatible state is never reset or adopted.
4. Before write-mode activation, stop and fence all archive writers and automatic
   restarts. Grant the daemon `CAP_SYS_ADMIN` using the supplied systemd unit
   or equivalent permissions. Preserve the fence through the matching-shim restart.
5. Start the daemon:

```bash
/usr/bin/objectveil /etc/objectveil/config.json
```

6. Require the configured mode/readiness, then start every managed `p4d` with the matching preload shim:

```bash
LD_PRELOAD=/path/to/libobjectveil_shim.so \
OBJECTVEIL_DEPOT=/mnt/perforce/depots/cod \
OBJECTVEIL_SOCK=/mnt/perforce/depots/cod/.objectveil/shim.sock \
OBJECTVEIL_ACCESS_SOCK=/mnt/perforce/depots/cod/.objectveil/access.sock \
p4d ...
```

The two socket variables are optional unless you want non-default socket paths.

## State Directory

A fresh default deployment keeps local state under `depot_path/.objectveil`.
An existing state remains at its verified current path; an explicit `state_dir`
continues to select its actual location. Do not move it without a verified state migration.
The selected directory contains:

- `manifest.redb`: persistent file manifest and dirty queue
- Terminal-failure ledger: existing daemon-owned JSONL log of upload failures; preserve its recorded filename
- `shim.sock`: Unix stream socket used by `libobjectveil_shim.so`
- `access.sock`: Unix datagram socket used by the access logger

This directory is reserved and never treated as depot content.

## Startup Scan

When `skip_startup_scan = false`, a writer walks the local depot in the
background to reconcile archives with its manifest.

- On a first bootstrap with an empty manifest, those files are queued for upload.
- On later restarts, untracked files are also queued as Dirty for upload. Tracked
  Clean files whose size or content changed are marked Dirty again. Upload and
  eviction workers wait for successful reconciliation; a failed scan keeps those
  workers gated while the daemon retries.

Startup also does manifest recovery in the background now, using cached summary
metadata for immediate readiness and then reconciling any stale `Uploading`
entries after the daemon is already live.

Access-log sink setup is also best-effort and happens after the cache core is
already up, so a slow or unavailable Elasticsearch/Postgres sink no longer
delays shim availability.

On very large populated caches, that background disk walk can consume I/O.
Leave `skip_startup_scan = false` on writers unless independent controls establish
that no archive was created or changed without being durably recorded. Skipping
the scan can leave unrecorded writes unpublished and manifest metadata stale.
Eviction still checks the locked local file's size and hash before removing it;
that final guard does not publish missed writes. Read-only caches do not run this scan.

Socket creation alone is not readiness: the listener can exist during license,
manifest, or cloud-key checks. Before starting dependent Perforce operations,
use a client implementing the bundled shim's versioned role handshake against
the selected shim socket and require the configured writer or read-only role. Bootstrap errors are temporary failures, not
permission to bypass the shim. Readiness does not mean the background writer
reconciliation or upload backlog has finished; check their status separately.

## Offline Manifest Summary Builder

`objectveil-manifest-summary` precomputes the manifest summary that the daemon uses
at startup. This is an offline maintenance utility for initialized native state,
not a format converter.

Run it with the daemon stopped:

```bash
systemctl stop objectveil
/usr/bin/objectveil-manifest-summary /etc/objectveil/config.json
systemctl start objectveil
```

The command rewrites only the manifest summary metadata under
`depot_path/.objectveil/manifest.redb`. It also recovers any files left in
`Uploading` state back to `Dirty`, the same way daemon startup does.

## Disaster Recovery (chunk packing)

Packing is always on. Recovery needs the primary payloads, catalog history,
permanent writer tickets, standalone descriptors, current keys and poison history,
plus independent Perforce checkpoints/journals. The local catalog is rebuildable
when remote identity is unambiguous. Fence the old writer and keep the replacement
daemon stopped while reconstructing:

```bash
systemctl stop objectveil
/usr/bin/objectveil-rebuild-catalog /etc/objectveil/config.json
```

The utility reconstructs mappings from chunk footers and standalone descriptors
and re-emits missing catalog records. Incomplete or ambiguous recovery exits
nonzero and leaves a durable startup/cleanup latch. It preserves payloads, but
must run with writers fenced. Do not restart serving until independent archive
verification, including actual `p4 verify`, establishes the recovery point.

See [`disaster-recovery.md`](disaster-recovery.md) for the full recovery
procedure and the acknowledged-submit loss window before asynchronous upload.

## Obliterate (compliance delete)

When p4d runs `p4 obliterate` it removes the archive file; the shim's
delete hook forwards that to the daemon, which **crypto-shreds** the content:
in encrypted deployments each archive uses a per-path key epoch. Revoking that
key makes its ciphertext copies undecodable, provided no recoverable old key
versions exist. Plaintext NFS deployments do not provide cryptographic deletion.
Under a drain lock, the daemon:

- persists a deletion intent for the selected epoch, refuses unresolved eviction
  journals, and unlinks the local plaintext file with a parent-directory sync;
- **shreds** the per-file key `keys/{storage_key}` by OVERWRITING it with a
  poison marker in one atomic PUT (not a plain delete), and deletes the legacy
  path-addressed object when present; and
- writes the durable cross-daemon Remove tombstone and removes local mappings
  before clearing the intent and acknowledging success.

Required durability failures leave deletion retryable. Intent replay stays pinned
to the old epoch and preserves newer re-added data. Immutable standalone payloads
and descriptors can remain physically present; encrypted old copies lose their
key, while plaintext NFS history can remain recoverable. New read-only opens check
remote authority, but already-open file descriptors keep snapshot semantics.

The overwrite (vs a delete) also closes a rare race: a plain delete leaves the
key name momentarily absent, so an ingest that was reading the same revision as
the obliterate ran could re-mint a usable key into the empty slot. The atomic
poison-overwrite means the slot is never empty, so a straggling ingest fails
closed instead. If the shred fails (a backend blip), the daemon returns an error
and p4d **retries** — the operation is idempotent.

> **Compliance-critical deployment precondition — the keystore bucket MUST NOT
> have versioning.** Poison-overwriting a key only shreds its ciphertext if no
> recoverable prior key version remains. Provision the **keystore bucket**
> (`keys/`) with **object versioning OFF, soft-delete / recycle-bin OFF,
> object-lock / WORM OFF, and no backup or cross-region replication.** On a
> *versioned* keystore bucket the prior key version survives the shred —
> recoverable, so the shred is defeated — while a current-version read *looks*
> shredded. Startup probes check current-object overwrite/delete behavior and
> create-once support. They cannot prove hidden retention or historical backups
> are absent. On **S3 and Azure** the daemon additionally
> detects a *versioned* keystore bucket directly at startup — a canary PUT that
> comes back with a version-id proves the bucket has versioning enabled — and
> **refuses to start**. On **GCS** that signal does not exist (every object
> carries a generation regardless of versioning), so a non-versioned GCS keystore
> bucket remains a provisioning requirement the daemon cannot verify. Because it
> cannot verify it, a GCS keystore config **must set
> `gcs_versioning_disabled_confirmed: true`** — the operator attesting the bucket
> has no versioning/soft-delete/retention; the loader refuses to start a GCS
> keystore without it. (The DATA bucket may keep versioning for durability —
> only the keystore bucket must not.)

Operator notes:

- A **re-add** of an obliterated depot path is served normally. It is minted
  under a **new per-path key epoch** with a fresh encryption key, so the
  re-added revision is readable while the obliterated prior revision's key
  stays poisoned and its ciphertext stays inert. The re-added content may be
  byte-identical to the obliterated file or entirely new — either way the
  re-add succeeds and never resurrects the obliterated key. (Writes captured
  at an epoch older than the current one fail closed, so a straggling ingest
  that raced the obliterate cannot re-mint the poisoned key.)
- An NFS primary (dev/test) has no keystore and no encryption, so obliterate
  there is best-effort local cleanup only — it is not a compliance target.
- `objectveil-rescan` (below) is the audit backstop.

Design detail: [`design/obliterate-crypto-shred.md`](design/obliterate-crypto-shred.md).

## Cache Integrity Rescan

`objectveil-rescan` is a standalone operator tool that cross-references the
cache against p4d's `db.storage` — the authoritative record of which
archive revisions still exist — and surfaces drift:

- **Orphans** — a cached blob whose `(depot_path, lbr_rev)` has **no**
  `db.storage` row. p4d obliterated the revision; the daemon normally
  *destroys* the cached bytes as part of the obliterate (see **Obliterate
  (compliance delete)** above), so an orphan here means that destruction
  did not reach or complete on this cache — a crash mid-obliterate, or the
  shim could not reach the daemon. The rescan is the backstop that
  surfaces it.
- **Mismatches** (only under `--verify-digest`) — the recomputed archive
  MD5 disagrees with `db.storage.digest`, or the on-disk gzip/raw shape
  disagrees with what the storage row's `lbrType` implies.

```bash
# Report only (no changes), against the running p4d:
/usr/bin/objectveil-rescan /etc/objectveil/config.json --dry-run \
  -p perforce:1666 -u objectveil

# After stopping/fencing the cache daemon and every archive writer:
# Query a stable Perforce metadata service or maintenance snapshot.
/usr/bin/objectveil-rescan /etc/objectveil/config.json \
  --offline-quarantine --quarantine-dir /var/lib/objectveil/quarantine

# Deep content audit (heavy — one decompress + MD5 per file):
/usr/bin/objectveil-rescan /etc/objectveil/config.json --verify-digest --dry-run
```

Flags: `<config.json>` (positional), `--dry-run` (report, act on
nothing), `--verify-digest` (re-hash every file), `--quarantine-dir
<path>`, `--offline-quarantine`, and the p4 connection flags `--p4-bin` / `-p`/`--p4-port` /
`-u`/`--p4-user` (default `p4` + the usual `P4PORT`/`P4USER` env). Depot
layout is derived entirely from `depot_path` in the config — the tool
does **not** query p4d for a depot map, only for `db.storage`.

**Quarantine semantics.** Use a dedicated private quarantine directory. A new
one is created `0700`; an existing one must already be owner-only and either
empty or carry the exact deployment/depot ownership marker. Unrelated or
symlinked directories are refused without changing their permissions. Bounded
filenames identify quarantined objects, and private durable source receipts
preserve the original archive paths for triage. Quarantining is a filesystem
**move** of a standalone file — the tool **never** hard-deletes cached data. With **no**
`--quarantine-dir`, drift is only reported and every file is left in
place. `--dry-run` acts on nothing regardless. A `db.storage` lookup
error (p4d unreachable, depot query failed) always leaves the affected
files in place and counts `lookup_errors` — it never quarantines on an
errored or malformed lookup. Moving files requires `--offline-quarantine`,
which explicitly attests that every archive writer is stopped or fenced for
the entire run. The tool also holds exclusive access to the existing manifest
to exclude the cache daemon, locks each candidate, and refuses a changed inode
or occupied destination. Keep Perforce metadata stable during the scan. A live
scan must use report-only mode; an attestation cannot itself fence an unmanaged
writer. After a file is quarantined the daemon re-detects it as
a cache miss on the next access and restores it from the backend.

**`--verify-digest` cost.** By default a file with a matching storage row
is counted `healthy` without reading its bytes. `--verify-digest` reads
and hashes **every** file (decompressing gzipped archives first) — one
full decompress + MD5 per file — so reserve it for a periodic deep audit,
not a routine sweep.

**Catalog scan caveat.** The tool scans local archive files and catalog pointers
to packed members or immutable standalone objects. Only local files can be
quarantined. Remote catalog orphans/mismatches are reported without moving or
deleting their objects; the existing `packed_orphans` / `packed_mismatches`
counters include immutable standalone pointers too. Packed verification reads
members through the authenticated chunk reader; immutable standalone verification
checks its address, content and required key. A report does not itself authorize
garbage collection. Compaction publishes replacement chunks but retains old
chunk payloads, seal markers and recovery metadata. Physical chunk and unknown-blob
sweeps issue no deletion or inventory requests: timed-out deletes could otherwise
remove later publications at a reused content address. Obsolete immutable
standalone versions/descriptors also remain remotely; plan storage for retained
history. This does not disable local cache eviction.

For report-only scans the manifest is opened **read-only** (a `redb`
`ReadOnlyDatabase`). Quarantine holds an exclusive existing-database handle
without migrations or catalog changes, and refuses a missing or locked manifest.
`redb` still forbids opening a read-only handle while a read-write
`Database` is open, so if the daemon is running the remote catalog
scan is **skipped** — the tool logs a warning and reports
`packed_scan_skipped=true`, but does **not** abort: standalone-file drift
detection still runs in full, with incomplete exit status. Stop the daemon (or
point the tool at a manifest snapshot) to include packed and immutable standalone
pointers in the scan.

The summary line reports `files_scanned`, `healthy`, `orphans`,
`packed_orphans`, `mismatches`, `packed_mismatches`, `path_decode_failed`
(files that don't decode into the `,d/<lbrRev>[.gz]` archive shape —
operator cruft, RCS `,v` files, temp files), `lookup_errors`,
`quarantined`, `verify_errors`, `drift_action_failed`,
`quarantine_collisions` (a drifted file left in place because the
quarantine destination was already occupied), and `packed_scan_skipped`.
Exit codes:
`0` = complete with no unresolved drift, `1` = setup error, `2` = unresolved
drift in either report-only or action mode, `3` = the scan was
**incomplete or remediation failed** — a `db.storage` lookup errored, a
content verify errored, the packed scan was skipped, or a drift quarantine move failed. A wrapper
must treat `3` as distinct from `0`: some files were never classified,
or drift was found but NOT successfully acted on.

## Logging

- `verbose = true` changes the default log level to `debug`.
- `RUST_LOG` overrides the default filter completely.
- `log_file` redirects stdout/stderr before tracing initializes.
- `stats_interval_secs` controls the periodic daemon stats line. Set it to `0` to disable the periodic stats logger.

Example:

```bash
RUST_LOG=info /usr/bin/objectveil /etc/objectveil/config.json
```

## Backgrounding

`daemon = true` detaches the process with `libc::daemon(1, 1)` before Tokio starts.

Operational notes:

- If `log_file` is set, daemon logs go there.
- If `daemon = true` and `log_file` is unset, stdout/stderr are redirected to `/dev/null`.
- `pid_file` is still the cleanest way to track the detached process.
- `daemon = true` is for standalone launches only. Under `systemd`, keep `daemon = false`.
- `objectveil` now rejects `daemon = true` when it detects a `systemd` service environment, because the packaged unit is `Type=simple` and should not fork.

Use a process supervisor such as systemd, runit, or supervisord for production restarts and lifecycle management.

## Metrics

When `metrics_file` is set, the daemon periodically writes a Prometheus textfile. This is intended for node_exporter textfile collection.

Example:

```json
{
  "metrics_file": "/var/lib/node_exporter/textfile_collector/objectveil.prom",
  "metrics_interval_secs": 15
}
```

### Cost and backend-health signals

Beyond the semantic counters (`objectveil_uploads_total`,
`objectveil_chunk_gets_total`, …), two families make object-store **request
cost** and **primary health** directly observable:

- `objectveil_backend_requests_total{class="get|head|list|put|delete"}` — every
  billable request the cloud backend issues, bucketed by request class. `list`
  is usually the surprise cost driver (whole-namespace scans by backfill,
  orphan GC, and rebuild); watch its rate to confirm the backfill idle back-off
  is keeping a caught-up daemon quiet. A multipart upload counts as one `put`
  (its parts are not separately counted).
- `objectveil_backend_bytes_total{direction="downloaded|uploaded"}` — bytes moved
  each way, counted on successful transfer. Pair with the request counts to see
  average object size and egress volume.
- `objectveil_secondary_fallbacks_total` — reads the **primary** did not have
  (NotFound) that were served off the read-only **secondary**. A rising rate
  means the primary is missing objects (degraded, mid-outage, or lagging an
  NFS→cloud migration) even while restores still succeed. Alert on any sustained
  increase.

At startup the daemon also probes the primary once and logs
`primary backend reachable at startup` or a `primary backend probe FAILED`
**warning** — a misconfigured bucket/region, bad credentials, or a network
partition shows up in the log immediately instead of on the first user read.

## Access Logging

Access logging requires both:

- `access_log_enabled = true` (the default)
- exactly one of:
  - `access_es_url` set to an Elasticsearch base URL
  - `access_pg_url` set to a PostgreSQL connection string

The default of `access_log_enabled = true` means a config that omits
both sinks is **rejected at load time**. If access logging is not
desired, set `access_log_enabled = false` explicitly.

(Earlier docs described "best-effort disabled" behavior here; the
loader's no-sink rejection makes that path explicit.)

`access_es_auth` uses `user:pass` basic auth format. `access_es_ca_cert` should point to a PEM CA bundle if Elasticsearch uses a private CA.

For PostgreSQL:

- put credentials and any desired `sslmode` in `access_pg_url`
- use `access_pg_table` to select the destination table, optionally schema-qualified
- use `access_pg_ca_cert` for private TLS roots
- use `access_spool_max_bytes` if you want bounded on-disk buffering during sink outages

Access-log failure handling:

- repeated failure warnings are rate-limited by `access_failure_warn_interval_secs`
- remote writes retry with exponential backoff between `access_failure_backoff_initial_secs` and `access_failure_backoff_max_secs`
- the sink auto-disables after `access_disable_after_failures` consecutive failures unless that setting is `0`
- when `access_spool_max_bytes` is set, failed batches are spooled on disk up to that hard cap
- when the spool is full or unset, undeliverable access batches are dropped

## Backend Guidance

### Azure

- Use either `sas_token`, `account_key`, or the Azure SDK default credential chain.
- `endpoint` is optional. If omitted, the daemon uses `https://<account_name>.blob.core.windows.net/<container>`.
- `ca_cert_path` and `verify_ssl: false` are **rejected at config load** (see `docs/CONFIGURATION.md`'s Backend Object table) — the `object_store`-backed builders don't honour either, so the loader fails closed rather than silently ignoring them. There is no working custom-CA/relaxed-TLS override today.
- `restore_chunk_threshold_mb` and `restore_chunk_threads` apply to Azure, S3, and GCS large-file restores. NFS ignores both. Tune them once per cloud-backend choice; they aren't Azure-specific.

### S3

- `bucket` is required.
- `endpoint` is optional and is how you target S3-compatible services such as MinIO.
- If you omit `access_key` and `secret_key`, the AWS SDK default credential chain is used.
- `ca_cert_path` and `server_side_encryption: true` are **rejected at config load** — neither is wired through the `object_store` S3 builder. Configure default bucket encryption on the AWS side instead of `server_side_encryption`.
- `verify_ssl = false` is rejected at config load, with no working fallback (not even via `ca_cert_path`, which is also rejected) — install the endpoint's CA in the system trust store instead.
- `region` should match the region the daemon runs in. Co-locate the bucket
  and the daemon: a cross-region bucket bills inter-region data transfer on
  every restore and upload. On EC2, if `region` is set (and no custom
  `endpoint` is configured) the daemon compares it against the instance's own
  region at startup via IMDSv2 and logs a **warning** on a mismatch — advisory
  only, but a cheap way to catch an expensive misconfiguration.

#### S3 credential refresh

The SDK resolves credentials via the standard AWS provider chain. How
long the daemon keeps working without operator intervention depends on
which provider fires:

- **IAM instance role, IRSA, EKS Pod Identity, ECS task role.**
  Credentials auto-refresh in the background; no action required.
- **Static `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` (no session
  token).** Long-lived credentials — no expiry, no refresh needed.
- **Static `AWS_SESSION_TOKEN` in the daemon environment.** The STS
  token has a fixed TTL (typically 1–12 h). Once it expires, every
  upload/restore will get a 403 and the daemon does **not** self-
  recover: it reads the environment once at process start. Restart
  the daemon with a fresh token, or — preferred — migrate to an IAM
  role so refresh is transparent.

Operators watching for this condition should alert on a sustained
`objectveil_uploads_total{result="failure"}` increment paired with repeated
403s in the daemon log.

### NFS

- `path` must already exist.
- `secondary` is useful when you want a restore fallback to another local filesystem — it is only ever *read* by the restore path (uploads always go to `primary`), though it can still be written to for deletes/reconciliation during obliterate and backfill.

### GCS

- `bucket` is required. `credentials_file` is optional when Application Default Credentials are available.
- With `credentials_file`, the backend signs a JWT locally from the service-account key and uses it directly as the bearer token (`object_store`'s `SelfSignedJwt`) — there is no token-endpoint round trip to exchange it for a separate OAuth token.
- `project_id` is accepted but currently **dead** — it's parsed into the config and never sent to the GCS client (no `x-goog-user-project` header). Safe to omit.
- GCS `endpoint` overrides are rejected at config load. Select the storage bucket with `bucket`; a `gs://` endpoint could otherwise replace that bucket and defeat the separation between archive data and keys.
- `ca_cert_path` and `verify_ssl: false` are **rejected at config load**, the same unconditional rule as S3/Azure — not something specific to the GCS SDK.

### Storage cost & tiering

The daemon uploads every object in the **Standard** storage class and does not
pin a colder class at write time. That is deliberate: a cache cannot tell hot
from cold when it writes an object (every chunk is written once and read later),
so tiering belongs in a **bucket lifecycle rule** that demotes objects by access
age — which only moves data that actually goes cold.

Recommended lifecycle configuration on the **data** bucket:

- An **Intelligent-Tiering** rule (or an IA transition) scoped to the `chunks/`
  prefix. This is the single largest bytes-at-rest cost lever for cold archive
  history.
- **Exclude the `keys/` prefix** — the crypto-shred keystore must have no
  lifecycle, versioning, soft-delete, or object-lock, so that an obliterate's
  key delete is final (see the keystore requirements). If the keystore lives in
  its own bucket (recommended), scope lifecycle to the data bucket only.
- **Avoid Glacier / non-instant-retrieval tiers** on `chunks/` — they are on the
  restore-serving path and would turn a cold read into a multi-hour restore.
- Keep an **AbortIncompleteMultipartUpload** rule (1–7 days) on the data prefixes
  as the backstop for parts orphaned by a crash mid-multipart.

Watch `objectveil_backend_bytes_total` and `objectveil_backend_requests_total` (see
*Metrics*) to size the transfer and request volume a tiering change will affect.

## Recommended Rollout

1. Use a separate read-only cache to verify restores from an already populated backend. Do not switch an active submit server to read-only mode for this drill.
2. Turn on `metrics_file` and confirm the textfile exporter is producing sane counters and gauges.
3. If you need access analytics, enable Elasticsearch or PostgreSQL logging after restore behavior is stable.
4. Enable write mode on the writer after the daemon has `CAP_SYS_ADMIN` and fanotify starts cleanly. Restart p4d with the updated shim when deploying or changing roles.

## Schema & Protocol Versioning

`objectveil` carries independently versioned formats. The safety changes described
in [disaster recovery](disaster-recovery.md) require a coordinated fleet upgrade:
fence old readers/writers before publishing the new catalog and standalone format.
Mixed native and incompatible formats are refused; use verified conversion for format transitions.

### 1. Shim ↔ daemon socket protocol

Requests use an internal versioned prefix, followed by an operation and its
relative path or request token. Use the matching bundled shim to construct
requests; do not derive a wire prefix from the executable name. Operations
include FETCH, STAT, STATSTART, STATPOLL, DELETE, MODE and ENSURE. The protocol
prefix is exactly `OBJECTVEIL/1 `.

Unprefixed requests are rejected. The current shim needs
MODE (`OK WRITER` or `OK READONLY`) before managed access. Unknown roles fail
closed. Each read-only open begins with ENSURE and uses FETCH only to continue
its progress polling; it cannot reuse an earlier read's authority decision.

**FETCH is a progress-poll protocol, not one synchronous round trip.** A cold restore runs as a detached, single-flight task
on the daemon; a poll from the shim gets a `RESTORING\n` reply until
the file materializes, instead of the daemon blocking the RPC for the
whole restore. This is what lets a large standalone archive over a
slow/cross-cloud backend (multi-part ranged GET, easily past the
shim's `rpc_timeout`) restore successfully instead of the shim giving
up and p4d reporting "Error opening librarian file." An absolute
`OBJECTVEIL_RESTORE_MAX_WAIT_MS` backstop (default 5 min) still bounds a
daemon stuck in `RESTORING`; a silent or erroring daemon still fails
fast on the first transient.

**Authenticated metadata can also require a cold restore.** The current shim
starts a fresh `STATSTART` operation. Quick checks return ordinary metadata;
slower checks return `STATWAIT <request-token>`, and only that token can be
continued with `STATPOLL`. Each request validates independently, including two
requests for the same archive. The daemon bounds the total admitted operations
and retained results to 64, with at most 16 executing. Completed results expire
after 60 seconds and are consumed once. The shim's absolute
`OBJECTVEIL_RESTORE_MAX_WAIT_MS` deadline also bounds metadata polling; errors do
not restart the operation automatically.

Synchronous `STAT` is also supported. A shim falls back to it only on an exact
protocol-unsupported reply to `STATSTART`; that fallback retains the synchronous
RPC timeout. Deploy the matching daemon and restart shim-bearing processes
together to use progress polling.

**Writer inline short-wait (since 5.0.5).** The unconditional `RESTORING` on
the very first poll cost every cold read a full extra round trip
(measured ~+26 ms, ~+36% for small files) even when the restore itself
was fast. `fetch_poll` now does a bounded inline wait
(`SHIM_FETCH_INLINE_WAIT_MS`, 250 ms) before replying: a restore that
finishes within the wait returns `OK` in the *same* round trip as the
old synchronous behavior, and only a restore slower than 250 ms falls
through to `RESTORING` + polling. The shim's `OBJECTVEIL_RPC_TIMEOUT_MS`
override is floored at `MIN_RPC_TIMEOUT_MS` (1 s, compile-time
guaranteed to exceed the inline wait), so a too-aggressive timeout
override can't spuriously fast-fail during the inline wait.

Read-only validation registers its task immediately and uses progress polling
without the writer's inline wait. This prevents an older validation from being
reinserted after a newer read has started. Warm validation also hashes the local
file; account for that I/O and authority traffic when sizing reader caches.

**Observability.** `objectveil_restores_polled_total` counts cold restores
that outlasted the inline wait and were served via the `RESTORING` poll
path — i.e. slow restores (large files, or a slow/degraded/throttled
backend). Watch it against `objectveil_restores_total{result="success"}`:
a rising *fraction* means restores are getting slow — the condition the
progress poll exists to survive, and a cue to check backend health or
cache sizing before it becomes user-visible. (The periodic packing-stats
log line also reports it as `… N polled`.)

All fleet members must use the matching native protocol release. Fence writers,
stop relevant services and restart every shim-bearing application after installing
the complete release. On-disk library replacement does not replace mapped code.

### 2. Manifest FileEntry codec

Native FileEntry records begin with `OV` and an explicit version byte. The
reader supports native layouts 0 (fixed fields), 1 (optional content hash) and
2 (epoch identity in addition to the hash). Unknown versions, unversioned data
and JSON records are rejected. The namespace-bound state marker must be present
before populated state can be opened. Do not edit binary records or marker
files to bypass incompatible format errors; preserve the source for verified
conversion. Current writes retain explicit versions and authenticated recovery
identities.

### 3. Access log entries

`AccessEntry` carries a `schema_version` field (currently `1`). When
adding new fields, bump the version and tag the field with
`#[serde(default)]` so older records are still decodable. For
Elasticsearch, rotate the index suffix
(for example, explicitly configure `objectveil-access-v<N>-<date>`) when the version changes. For
Postgres, add a migration under
`packaging/postgres/migrations/`.

## Checkpoint Verification

Fresh implicit verifier jobs use `.objectveil-verify-<input>`; a sole recognized
existing job is reused and ambiguous dual jobs are refused.
Prefer an explicit verified existing `--state-dir` for an upgrade. Verifier
ownership/progress records remain tied to that job identity.

`objectveil-verify` scans Perforce checkpoints and journals against backend
storage to find missing archive objects. It writes restart state under
the `--state-dir` path (separate from the daemon's depot `.objectveil`
directory) so long scans can resume across restarts.

### State directory layout

```
<state-dir>/
  .lock                     advisory lock; one verifier run per state dir
  meta.json                 input kind + canonical checkpoint path
  sources/source-<digest>.json  per-source scan progress (one file per
                            checkpoint/journal source)
  targets.regular.txt       verify targets for regular archives
  targets.shelf.txt         verify targets for shelved archives
  verified.regular.ok.txt   verify-OK list, regular
  verified.shelf.ok.txt     verify-OK list, shelf
```

### Non-UTF-8 archive keys on cloud backends

Archive paths that contain non-UTF-8 bytes (rare on Linux but legal)
use the engine's compatible reversible byte-path encoding when checking S3,
Azure or GCS. Do not construct or rename these internal keys manually. This is the same encoding `objectveil` uses for uploads, so
objects written by the daemon verify correctly. If you're verifying
against a cloud bucket populated directly by p4d (e.g. a migration
seed) under literal byte-named keys, those will be reported as
missing. Re-upload through `objectveil` to reconcile.

### Unavailable objects

When a backend returns transient errors (5xx, 429, connection resets)
for a row past the retry budget (5 attempts with exponential backoff),
the row is recorded in `<state-dir>/unavailable_objects.txt` as a
JSON-lines log and reported under `unavailable_objects` in the
summary. The scan continues; these rows can be replayed after the
backend recovers by re-running the verifier. Non-transient errors
(auth failures, bucket-not-found) still abort the scan.

### Journal coverage

In journal mode, `objectveil-verify` processes `@pv@` (insert) and
`@rv@` (in-place replace — e.g. `p4 retype`, attribute digest
updates, edge-server catch-ups) rows and verifies their archives.
`@dv@` (delete) rows on `db.revsh` are used for cross-source shelf
deletion tracking but never generate their own verify targets.
Eligible revision rows with unknown actions, malformed fields or oversized
records fail the scan as incomplete; they are not silently skipped. Known
no-content actions remain excluded from archive targets.

### Per-source state and rotating symlinks

Resume state binds ordered source paths, exact bytes and filesystem identities,
plus effective Perforce/backend configuration and the native namespace UUID.
Changing the namespace invalidates saved progress; use `--reset-state` to run
a complete new verification. Progress created without that namespace binding
cannot be accepted as proof for the current namespace.

A rotated symlink, changed or
appended source, added split input or different target is rejected until the
operator supplies `--reset-state` for a new complete run. Use stable input
snapshots throughout both parsing and integrity phases. Old per-source state is
pruned only within a run whose initial identity validation has succeeded.

## Troubleshooting

### Cold restores do not happen

Check:

- `LD_PRELOAD` points to `libobjectveil_shim.so`
- `OBJECTVEIL_DEPOT` matches the local depot root exactly
- the daemon is running
- `.objectveil/shim.sock` exists and is reachable

### Uploads never start

Check:

- `read_only` is not set
- the daemon has `CAP_SYS_ADMIN`
- fanotify started successfully in the daemon log
- `primary` is a valid backend config

### Access logs never appear in the remote sink

Check:

- `access_log_enabled = true`
- exactly one of `access_es_url` or `access_pg_url` is set
- `.objectveil/access.sock` exists
- `OBJECTVEIL_ACCESS_SOCK` matches the daemon socket path if you overrode it

If you are using Elasticsearch, also check:

- `access_es_auth` if the cluster requires basic auth
- `access_es_ca_cert` if the cluster uses a private CA

If you are using PostgreSQL, also check:

- `access_pg_url` includes the right host, database, credentials, and `sslmode`
- `access_pg_table` points at a valid table name
- `access_pg_ca_cert` if PostgreSQL TLS uses a private CA
- `access_spool_max_bytes` and `access_spool_dir` if you are relying on the bounded spool during outages

### S3 config fails when `verify_ssl = false`

This is rejected intentionally. The current object-store configuration also
rejects `ca_cert_path`; it does not offer a custom trust-root or insecure
no-verification option. Use an endpoint certificate trusted by the configured
client, and keep certificate verification enabled.
