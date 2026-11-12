# Configuration

Upgrading an existing installation? Follow the
[ObjectVeil upgrade and format guide](upgrading.md) before changing
services, state directories, environment variables or cloud prefixes.


`objectveil` reads a single JSON config file:

```bash
objectveil /path/to/objectveil.json
```

Configuration requires `"format": "objectveil/1"` and a canonical, non-nil
`namespace_id` UUID. All clients sharing a data/key namespace must use the same
UUID. The namespace is immutable during reload. Templates deliberately contain
a nil UUID that must be replaced.

The default state path is `depot_path/.objectveil`. An explicit `state_dir` is
supported. Populated unmarked state is refused; it is never adopted by guessing
filenames. Before daemon startup, run `objectveil-initialize <config.json>` under
the archive-owner UID. Existing archive-only roots require the explicit
`--existing-archives` attestation. See [upgrading](upgrading.md).

New to `objectveil`? [`QUICKSTART.md`](QUICKSTART.md) is the narrative
walkthrough; this page is the field-by-field reference.

The full annotated reference file, showing every top-level and backend key:

- [`docs/examples/objectveil.full.example.jsonc`](examples/objectveil.full.example.jsonc)

For provider-specific templates with example thread/packing settings, use
one of the files below. Replace credentials, paths and the nil UUID, remove
JSONC comments before loading, and review the deployment guide. A configured
secondary must be an explicitly owned native backend with matching identity
and catalog/key dependencies; it is not an arbitrary read-only NFS fallback:

- [`docs/examples/objectveil.s3.example.jsonc`](examples/objectveil.s3.example.jsonc) (AWS S3)
- [`docs/examples/objectveil.azure.example.jsonc`](examples/objectveil.azure.example.jsonc) (Azure Blob Storage)
- [`docs/examples/objectveil.gcs.example.jsonc`](examples/objectveil.gcs.example.jsonc) (Google Cloud Storage)
- [`docs/examples/objectveil.nfs.postgres.example.jsonc`](examples/objectveil.nfs.postgres.example.jsonc) (NFS-only + Postgres access log)

## Output destinations

Log, PID and metrics paths must be distinct and must not overlap archives,
backend roots, the access spool, configuration, credentials or licenses.
Rotation destinations are checked too. Inside `state_dir`, use the standard
filenames (`objectveil.log` or `daemon.log`, `objectveil.pid` or `daemon.pid`,
`objectveil.prom` or `metrics.prom`) or a dedicated `outputs/` subdirectory;
other native control paths are reserved.

Output parents must be owned by root or the daemon UID and must not be writable
by other users or groups. Use a private subdirectory beneath shared temporary
storage; files directly in a shared temporary directory are refused. Existing
outputs must be owned regular files with one link. Symlinks, special files and
hard-linked outputs are refused before modification.

## Top-Level Options

| Key | Type | Default | Notes |
| --- | --- | --- | --- |
| `format` | string | required | Exactly `objectveil/1`. |
| `namespace_id` | UUID | required | Canonical non-nil UUID shared by this deployment; immutable on reload. |
| `depot_path` | path | required | Existing local depot root to manage. |
| `read_only` | bool | `false` | Disables fanotify, startup scan, and uploads. Cold restores still work through the shim socket. |
| `max_cache_gb` | u64 | `100` | Used only when `max_cache_bytes` is not set. |
| `max_cache_bytes` | u64 | unset | Overrides `max_cache_gb`. Must be `> 0`. |
| `low_watermark_gb` | u64 | unset | Low watermark in GiB. Mutually exclusive with `low_watermark_bytes`: setting both is rejected. |
| `low_watermark_bytes` | u64 | unset | Low watermark in bytes. Mutually exclusive with `low_watermark_gb`. If both are unset, the daemon uses `80%` of the effective max cache size. The effective value must be at least 1 MiB and no larger than the effective max cache size. |
| `upload_threads` | u16 | `8` | Write-mode upload coordinators. Must be `> 0` unless `read_only = true`. Runtime upload concurrency is `4 x upload_threads`. |
| `restore_threads` | u16 | `16` | Writer-mode restore workers; must be `> 0`. Read-only authority validation has a separate fixed admission limit of 16, including unfinished blocking hashes. Raising this setting does not raise that limit. |
| `restore_chunk_threshold_mb` | u64 | `8` | Cloud-backend large-restore threshold (Azure, S3, GCS). Files at or above this size switch to parallel range GETs when `restore_chunk_threads > 1`. `0` disables the threshold for all cloud backends. NFS ignores the field. |
| `restore_chunk_threads` | u16 | `8` | Cloud-backend parallel-range fan-out (Azure, S3, GCS). Must be `> 0` and `<= 64`; a value above `64` is rejected at load (not silently clamped). NFS ignores the field. |
| `primary` | object | required | Main storage backend. Uploads always go here. |
| `secondary` | object | unset | Optional restore fallback; not a mirrored write target. Normal deletion, backfill, and reconciliation can modify it, so it must be an explicitly owned backend, not an authoritative archive tree assumed to be read-only. See [OPERATIONS](OPERATIONS.md). |
| `keystore` | object | required for a cloud primary | Dedicated split-bucket backend holding the per-file crypto-shred keys (`keys/{storage_key}`). Must be a cloud backend (`azure`/`s3`/`gcs`) — `nfs` is rejected. **Mandatory whenever `primary` is a cloud backend**; the loader rejects a cloud primary that has no `keystore` (there is no single-bucket fallback). Must resolve to a **different bucket/container than `primary`** — a keystore pointing at the data bucket is rejected (a data bucket's versioning/soft-delete would preserve a shredded key). A **GCS** keystore additionally requires `gcs_versioning_disabled_confirmed: true` (see [GCS](#gcs)). Only an NFS primary (unencrypted dev/test) may omit it. |
| `skip_startup_scan` | bool | `false` | Skips writer reconciliation of untracked and changed local archives. Normally untracked files are queued as Dirty on every startup, and changed Clean files are marked Dirty before eviction is enabled. Leave false unless independent controls prove there were no unrecorded archive changes; skipping can leave writes unpublished and metadata stale. Eviction's final file identity/hash checks still apply. Ignored in `read_only` mode, where this scan does not run. |
| `stats_interval_secs` | u64 | `60` | **Hot-reloadable.** Periodic daemon stats log interval. `0` disables the periodic stats logger. |
| `daemon` | bool | `false` | When `true`, detaches with `libc::daemon(1, 1)` before Tokio starts. Prefer a service manager for production supervision. |
| `verbose` | bool | `false` | Changes the default log level from `info` to `debug` unless `RUST_LOG` is already set. |
| `pid_file` | path | unset | If set, the daemon writes its PID on startup and removes the file on clean shutdown. |
| `log_file` | path | unset | Redirects stdout/stderr to this file before tracing initializes. In daemon mode, omit only if you intentionally want logs discarded. |
| `metrics_file` | path | unset | If set, writes Prometheus textfile metrics here. Parent directories are created automatically. |
| `metrics_interval_secs` | u64 | `15` | Prometheus textfile export interval. Must be `> 0`. |
| `access_log_enabled` | bool | `true` | Enables the access-log pipeline. When `true`, exactly one remote sink (`access_es_url` or `access_pg_url`) must be configured or the config is rejected; set `false` to run without access logging. |
| `access_batch_size` | u64 | `10000` | Max deduplicated events per remote flush. Must be `> 0` and `<= 1,000,000`. |
| `access_max_pending_entries` | u64 | `200000` | Size of the in-memory channel between the access socket and the remote sink writer. Must be `>= access_batch_size` and `<= 1,000,000`; the finite bound protects the in-memory event channel. |
| `access_flush_interval_secs` | u64 | `5` | Access-log flush interval. Must be `> 0`. |
| `access_es_url` | string | unset | Elasticsearch base URL (`http://` or `https://` with a host). `http://` is rejected when `access_es_auth` or credentials embedded in the URL are present. Set this or `access_pg_url`, not both, when access logging is enabled. |
| `access_es_index` | string | `objectveil-access` | Elasticsearch index name for access records. ASCII alphanumerics, `-`, `_` and `.` only; must not start with `_` or `.`. Preserve the selected destination on upgrade. |
| `access_es_auth` | string | unset | Basic auth in `user:pass` form. |
| `access_es_ca_cert` | path | unset | PEM CA bundle for the Elasticsearch client. |
| `access_pg_url` | string | unset | PostgreSQL connection string (`postgres://` or `postgresql://`). It must carry an explicit `sslmode` of `require`, `verify-ca` or `verify-full`; a missing or weaker mode is rejected, and so is `require` when `access_pg_ca_cert` is set. Set this or `access_es_url`, not both, when access logging is enabled. |
| `access_pg_table` | string | `objectveil_access` | PostgreSQL table for access records, optionally schema-qualified (for example `ops.objectveil_access`). Each segment must match `[A-Za-z_][A-Za-z0-9_]*` and is quoted exactly as written, so the name is case-sensitive. The daemon creates the table when it is missing but never a schema: create the schema first and let the connecting role create tables in it, or create the table yourself. Preserve the existing destination on upgrade. |
| `access_pg_ca_cert` | path | unset | PEM CA bundle used as the only trust root for PostgreSQL TLS. Without it the daemon verifies the server against its bundled public root set, not the host trust store. The certificate is always verified, including with `sslmode=require`. |
| `access_failure_warn_interval_secs` | u64 | `60` | Minimum interval between repeated access-log failure warnings. Must be `> 0`. |
| `access_failure_backoff_initial_secs` | u64 | `1` | Initial retry backoff after a remote sink write failure. Must be `> 0`. |
| `access_failure_backoff_max_secs` | u64 | `60` | Maximum retry backoff after repeated remote sink write failures. Must be `> 0` and `>= access_failure_backoff_initial_secs`. |
| `access_disable_after_failures` | u32 | `100` | Auto-disables the remote sink after this many consecutive failures. `0` disables auto-disable and retries forever. |
| `access_spool_dir` | path | unset | Dedicated private directory for undelivered access-log batches. Defaults to `state_dir/access-spool` when `access_spool_max_bytes` is set. Must not contain symlinks or overlap archives, other state, or local backends; its parent must already exist. |
| `access_spool_max_bytes` | u64 | unset | Hard cap for the optional on-disk spool. When full, new undeliverable batches are dropped. Must be `> 0` when set. |
| `shim_socket_mode` | u32 | `432` | File mode applied to `.objectveil/shim.sock`. `432` is decimal for `0660`, which is how it must be written in JSON. Must be `<= 0770` with no world-access bits (`mode & 0o007 == 0`); a decimal value that looks like an octal file mode typed by mistake is rejected. |
| `access_socket_mode` | u32 | `432` | File mode applied to `.objectveil/access.sock`. `432` is decimal for `0660`, which is how it must be written in JSON. Must be `<= 0770` with no world-access bits (`mode & 0o007 == 0`); a decimal value that looks like an octal file mode typed by mistake is rejected. |
| `state_dir` | path | `depot_path/.objectveil` | Override for the daemon's internal state directory (manifest, sockets, spool, breadcrumbs). Existing state must have the exact native deployment marker; unmarked populated state is refused. The loader normalizes the path and resolves existing ancestors, rejecting unsafe overlap with backend roots or an ancestor of the depot. Internal state is excluded from archive discovery. Most operators leave this unset; the override exists for read-only depot mounts where the daemon needs to write state somewhere else. |
| `log_rotate_max_mb` | u64 | unset | Convenience input for `log_rotate_max_bytes` (multiplied by `1<<20`). Used only when `log_rotate_max_bytes` is not set. |
| `log_rotate_max_bytes` | u64 | unset | Live-log size threshold for rotation of `log_file` and the failed-uploads log. Unset disables size-based rotation. The main log rotates only when a size threshold is configured; the separate failed-uploads log also has daily age rotation. |
| `log_rotate_keep` | u16 | `5` | Number of historical rotated copies to retain (`.1` … `.N`). Older copies are removed by the rotation pass. |
| `failed_uploads_log_rotate_keep` | u16 | `10` | Per-day rotation count for the terminal upload-failure ledger. The daemon also rotates this file daily by mtime regardless of `log_rotate_max_bytes`. |
| `shim_allowed_uids` | u32 array | `[daemon_euid]` | Whitelist for `SO_PEERCRED` checks on the shim socket. Defaults to the daemon's own effective UID. Add the UIDs of every connecting process. This authorizes socket requests, not cross-UID archive serving: restored files remain daemon-owned and private. Filesystem permissions and peer checks also apply; see [socket access across users](OPERATIONS.md#socket-access-across-linux-users). |
| `access_allowed_uids` | u32 array | `[daemon_euid]` | Whitelist for `SCM_CREDENTIALS` on the access-log socket. Datagrams from outside the list are dropped at the receiver. |
| `watcher_allowed_uids` | u32 array | `[daemon_euid]` | Whitelist for fanotify writer authentication. CLOSE_WRITE events from non-allowlisted UIDs are dropped (counter: `objectveil_watcher_rejected_writer_total`). |
| `keep_caps` | bool | `false` | When `false`, the daemon drops `CAP_SYS_ADMIN` from its effective + permitted sets after fanotify is initialized. Set to `true` only when an external supervisor needs the capability to remain. The drop applies to the calling thread and sets `no_new_privs`; a failed drop is fatal rather than leaving the capability in place. |
| `max_reported_file_bytes` | u64 | `68719476736` | Largest file size the daemon will report to the shim in a FETCH/ENSURE or STAT reply (default is `MAX_REPORTED_FILE_BYTES`, 64 GiB). A larger archive is not served: the daemon logs an implausible-size error and answers with a backend-integrity error, which the shim surfaces as a retryable failure, and migration refuses to import it. To serve larger archives raise this setting and the shim's `OBJECTVEIL_MAX_FILE_BYTES` together. Must be `> 0`. |
| `enforce_content_hash` | bool | `false` | When `true`, restores fail closed if the post-download content hash does not match the manifest. This flag also refuses a restore when a previous local manifest entry lacks a stored hash. Native catalog restores always validate their authoritative content identity, regardless of this flag; uncatalogued raw objects are not a restore fallback. |
| `packing_enabled` | — | — | **REMOVED.** Cold-read chunk packing (packing many small archive revisions into content-addressed `chunks/<hex>` objects) is always on now; it is governed solely by `pack_max_file_bytes` (files below it pack, larger stay standalone). A config that still names `packing_enabled` loads with a deprecation warning and the field is ignored. Design: [`design/cold-read-chunk-packing.md`](design/cold-read-chunk-packing.md). |
| `dedup_enabled` | bool | `false` | Deduplicate eligible standalone uploads using shared encrypted blobs. Existing dedup objects remain readable with the flag off. Mutable paths whose serving key differs from the shared key use ordinary immutable standalone storage. Changes deletion semantics: shared-content destruction waits for background GC. See [enablement](enabling-content-dedup.md). |
| `dedup_gc_single_writer_ack` | bool | `false` | Operator attestation that this is the only writer for the backend. Permits known-content GC only with dedup enabled and complete remote-reference proof. This is not a distributed lock. Unknown orphan blobs are retained. Each GC pass that has something to prove reads the complete remote catalog and keystore once; see [cost and timing](enabling-content-dedup.md#garbage-collection-cost-and-timing). |
| `pack_target_chunk_bytes` | u64 | `67108864` | Target sealed-chunk size. An open L0 packer seals once it reaches this many compressed bytes. Must be `> 0`. |
| `pack_max_file_bytes` | u64 | `8388608` | Standalone bypass cutoff. Files at or above this size skip packing and upload/restore as standalone objects (bounding over-fetch). Must be `> 0` and `<= 1073741824` (the 1 GiB packed-reader limit). In-memory packing capture is separately capped at 16 MiB, so the effective cutoff is the smaller of the two. |
| `pack_flush_deadline_secs` | u64 | `30` | Flush deadline for a partially-filled open packer, so a trickle of submits still lands durably without waiting for the size target. Must be `> 0`. |
| `compaction_live_fraction` | f64 | `0.5` | A chunk with live members becomes a repacking candidate once its live-byte fraction (`live_bytes / total_bytes`) falls below this. Must be in `(0.0, 1.0]`. Old chunks and recovery metadata are retained; this does not reclaim remote bytes. |
| `backfill_enabled` | bool | `false` | Opt-in batch repacker for eligible, explicitly prepared path-addressed objects inside the native namespace. It walks only unreserved keys, so native `standalone/<id>` payloads, chunks, blobs, catalog and keys are never candidates. Gated behind this flag; with it off the repacker loop is never spawned. See [enablement](enabling-chunk-packing.md). |
| `backfill_rate` | u64 | `50` | Backfill pacing in objects/second (per-object delay = `1 / backfill_rate`). Must be `> 0`. Inert unless `backfill_enabled` is `true`. |
| `backfill_prefix` | string | unset | Restricts the backfill walk to storage keys under this depot-path prefix. Unset walks the whole object space while excluding all reserved internal namespaces and authoritative dedup objects. Inert unless backfill is enabled. |

## Hot Reload

The daemon polls its config file's mtime every ~30 s and re-reads on
change. The reloaded config is applied field-by-field. Saving the
file is enough — there's no signal required. Editors that swap-and-
rename (vim, `sed -i`, etc.) work because the daemon polls the path,
not the inode. A parse or validation failure keeps the previous
known-good config and logs a warning; a half-applied bad
config can't poison the daemon.

Sending `SIGHUP` to the daemon triggers an immediate re-check on top
of the poll, so an edit-then-signal workflow doesn't wait out the
~30 s cadence. `SIGHUP` is otherwise ignored (it does not terminate
the daemon); if signal registration itself fails at startup, reloads
still work via the mtime poll alone.

### Hot-reloadable fields

These take effect on the next loop iteration of the relevant task,
typically within a few seconds:

- `stats_interval_secs`, `metrics_interval_secs` — periodic-task cadence
- `verbose` — flips the tracing filter between `info` and `debug`
  (subject to `RUST_LOG` override)
- `enforce_content_hash`
- `max_reported_file_bytes`
- `max_cache_bytes`, `low_watermark_bytes` (and the derived
  eviction target)
- `shim_allowed_uids`, `access_allowed_uids`,
  `watcher_allowed_uids`
- `access_failure_backoff_initial_secs`,
  `access_failure_backoff_max_secs`,
  `access_disable_after_failures`,
  `access_failure_warn_interval_secs`

### Restart-required fields

Changes to these are detected and listed in a warning-level log line
on each reload, but the running daemon won't pick them up until you
restart it:

- Identity / paths: `namespace_id`, `depot_path`, `state_dir`, `read_only`,
  `daemon`, `pid_file`, `keep_caps`, `skip_startup_scan`
- Sizing committed to runtime structures: `upload_threads`,
  `restore_threads`, `restore_chunk_threshold_mb`,
  `restore_chunk_threads`
- Cold-read chunk packing (captured at engine-construct / loop-spawn
  time): `pack_target_chunk_bytes`,
  `pack_max_file_bytes`, `pack_flush_deadline_secs`,
  `compaction_live_fraction`, `backfill_enabled`, `backfill_rate`,
  `backfill_prefix`
- Deduplication: `dedup_enabled`, `dedup_gc_single_writer_ack`
- Backend blocks: `primary`, `secondary`, `keystore` (type, endpoint,
  bucket/container, credentials, prefix, storage class)
- Access-log structural settings: `access_log_enabled`,
  `access_es_url`, `access_pg_url`, `access_pg_table`,
  `access_es_index`, `access_es_auth`, `access_es_ca_cert`,
  `access_pg_ca_cert`, `access_max_pending_entries`,
  `access_spool_dir`, `access_batch_size`,
  `access_flush_interval_secs`, `access_spool_max_bytes`
- Sockets and rotation: `shim_socket_mode`, `access_socket_mode`,
  `log_file`, `metrics_file`, `log_rotate_max_bytes`,
  `log_rotate_keep`, `failed_uploads_log_rotate_keep`

## Derived Values

- `state_dir` defaults to `depot_path/.objectveil` and uses the override above if set.
- `depot_path` is canonicalized on load, so symlinked depot roots such as `/p4/1/depots` are supported.
- If `low_watermark_*` is omitted, the low watermark defaults to `80%` of the effective max cache size.
- The eviction target is derived, not configurable: `max(1, low_watermark * 90%)`.
- For non-NFS backends, `path_prefix` defaults to the basename of `depot_path` when omitted.
  Example: `depot_path = "/p4/1/depots"` implies `path_prefix = "depots"`. Set
  `path_prefix` explicitly whenever several hosts must address one namespace.

## Backend Object

The `primary`, `secondary`, and `keystore` blocks all use the same object shape.

| Key | Type | Applies To | Notes |
| --- | --- | --- | --- |
| `type` | enum | all | One of `azure`, `s3`, `gcs`, `nfs`. |
| `endpoint` | string | azure, s3 | Custom API endpoint/base URL. Useful for private Azure endpoints or S3-compatible services such as MinIO. GCS overrides are rejected so an endpoint cannot silently replace the configured bucket. **Requires `endpoint_pin = true`** — the loader rejects a custom endpoint without the explicit opt-in to guard against accidental endpoint redirection. |
| `endpoint_pin` | bool | azure, s3 | Required when `endpoint` is set; must be `true`. Acts as the explicit "yes, I really mean to override the SDK default" toggle. Omitting it is the safe default — operators who didn't intend to set a custom endpoint get a clear startup error instead of silently talking to the wrong host. |
| `bucket` | string | s3, gcs | Required by the config schema for `s3` and `gcs`. |
| `container` | string | azure | Required by the config schema and runtime for Azure. |
| `region` | string | s3 | Passed to the S3 client when set. When omitted the client assumes `us-east-1`, so set it for a bucket in any other region. |
| `access_key` | string | s3 | Static S3 access key ID. Set it together with `secret_key`; one without the other is rejected. When both are omitted the client uses the EC2 instance profile through IMDS (see [S3](#s3)). |
| `secret_key` | string | s3 | Static S3 secret paired with `access_key`. |
| `account_name` | string | azure | Required for an Azure backend. |
| `account_key` | string | azure | Optional Azure SharedKey credential. Mutually exclusive with `sas_token`. |
| `sas_token` | string | azure | Optional Azure SAS credential. Include either the raw query string or a leading `?`. When neither credential is set the client uses the host's managed identity (see [Azure](#azure)). |
| `project_id` | string | gcs | Accepted and carried through the config, but currently **dead**: it is never passed to the GCS client (no `x-goog-user-project` header is sent). Safe to omit. |
| `credentials_file` | path | gcs | Optional service account JSON key file. When omitted, the client uses the gcloud application-default credentials file in the daemon user's home directory if one exists, otherwise the GCE/GKE metadata server (see [GCS](#gcs)). Rejected on an `s3` backend. |
| `path` | path | nfs | Required by the config schema and runtime for NFS. Must already exist as a directory. |
| `path_prefix` | string | all | Storage key prefix. `prefix` is accepted as an alias in the JSON input. For NFS this is validated as a safe relative path. |
| `ca_cert_path` | path | azure, s3, gcs | **Rejected at config load for every backend type.** The `object_store`-backed cloud builders don't honour a custom CA bundle; setting this field at all (any backend) fails validation rather than silently no-op'ing. Install the CA in the system trust store instead. |
| `verify_ssl` | bool | azure, s3, gcs | Defaults to `true` (accepted). **`false` is rejected at config load for every cloud backend** — the `object_store` builders always use the system trust store, so a `false` value would silently not take effect; the loader fails closed instead. There is no working override (not even via `ca_cert_path`, which is also rejected). |
| `server_side_encryption` | bool | s3 | Defaults to `false` (accepted). **`true` is rejected at config load** — SSE-S3 is not wired through the `object_store` S3 builder. Configure default bucket encryption on the AWS side instead. |
| `gcs_versioning_disabled_confirmed` | bool | gcs — **keystore only** | Operator attestation that the keystore bucket has no versioning, soft-delete or retention. A GCS `keystore` block must set it to `true` or the config is rejected; it is ignored on S3/Azure keystores, whose versioning is probed at startup. See [GCS](#gcs). |
| `storage_class` | string | s3, gcs, azure — **primary only** | Cloud storage class (tier) applied to **new** data uploads. Omit (the default) to use the bucket's default class — zero behaviour change. **Primary data backend only:** setting it on a `keystore` or `secondary` block is rejected at config load, and it is rejected on an `nfs` backend (no object tiers). Only **synchronous-retrieval** classes are allowed; **asynchronous** classes (S3 `GLACIER`/`DEEP_ARCHIVE`, Azure `Archive`) are **rejected** because objectveil serves cold reads synchronously and they would stall a read until a minutes-to-hours restore. Applies only to PUTs made after the setting takes effect — **existing objects keep their prior class**. See [Storage class (data tiering)](#storage-class-data-tiering) below and [`TUNING.md`](TUNING.md). |

## Backend-Specific Notes

### Azure

- The Azure backend requires:
  - `type = "azure"`
  - `container`
  - `account_name`
- Authentication can use one of:
  - `sas_token`
  - `account_key`
  - the host's managed identity (through the instance metadata service) when
    neither is set. The client is built from this file only: `AZURE_CLIENT_ID` /
    `AZURE_TENANT_ID` / `AZURE_CLIENT_SECRET`, workload-identity token files and
    an Azure CLI login are **not** read.
- `endpoint` is actively used. `ca_cert_path` and `verify_ssl: false` are **rejected at config load** (see the Backend Object table above) — there is no custom-CA or relaxed-TLS override today.
- **As a keystore:** a separate container does not isolate account-wide policies.
  Azure blob versioning and blob soft-delete are configured for the storage
  account. If the data account retains old blob versions or deleted blobs,
  place keys in a separate account whose policies satisfy the
  [key-deletion requirements](disaster-recovery.md#key-durability-and-cryptographic-deletion).
  See Azure's [data-protection overview](https://learn.microsoft.com/en-us/azure/storage/blobs/data-protection-overview).
- Large restores can use parallel range GETs (the same rule applies to S3 and GCS) when all of the following are true:
  - `restore_chunk_threshold_mb > 0`
  - `restore_chunk_threads > 1`
  - the object size is at or above the threshold

### S3

- The S3 backend requires:
  - `type = "s3"`
  - `bucket`
- `region`, `access_key`, `secret_key`, and `endpoint` are actively used.
- Credentials come from `access_key` + `secret_key`, or — when both are omitted —
  from the EC2 instance profile through IMDS. The client is built from this file
  only, so `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_SESSION_TOKEN`,
  shared credential profiles, IRSA web-identity variables and ECS/EKS container
  credential variables are **not** read. Temporary session tokens cannot be
  supplied statically.
- `ca_cert_path` and `server_side_encryption: true` are **rejected at config load** — neither is wired through the `object_store` S3 builder. Configure default bucket encryption on the AWS side instead of `server_side_encryption`.
- `verify_ssl = false` is rejected at config load. There is currently no working custom-CA/relaxed-TLS override for S3 — install the CA in the system trust store instead.

### NFS

- The current NFS backend requires:
  - `type = "nfs"`
  - `path`
- `path_prefix` is validated as a depot-relative safe path.
- Storage keys are also validated to reject absolute paths, traversal, and entries under `.objectveil`.

### GCS

- The GCS backend requires:
  - `type = "gcs"`
  - `bucket`
- Authentication can use either:
  - `credentials_file`
  - when it is omitted, the gcloud application-default credentials file under
    the daemon user's home directory (`~/.config/gcloud/application_default_credentials.json`)
    if present, otherwise the GCE/GKE metadata server. The
    `GOOGLE_APPLICATION_CREDENTIALS` environment variable is **not** read; name
    the key file in `credentials_file` instead.
- `path_prefix` is actively used. GCS `endpoint` overrides are rejected; select the bucket with `bucket`. `project_id` is accepted but currently dead (never sent to the GCS client).
- `verify_ssl = false` and `ca_cert_path` are **rejected at config load** (same unconditional rule as S3/Azure — see the Backend Object table above), not just by the GCS backend specifically.
- Authentication and retries are handled by the shared `object_store` client rather than custom HTTP logic.
- **As a keystore:** a GCS keystore must also set `gcs_versioning_disabled_confirmed = true`.
  The daemon's current `object_store` interface does not expose GCS bucket policy
  metadata, and a GCS generation is stamped regardless of versioning. The daemon
  therefore cannot verify that the keystore bucket has versioning off — and a
  versioned keystore silently defeats crypto-shred, because a poison-overwrite leaves
  the prior key generation recoverable. The flag is the operator attesting the bucket
  has **no** versioning / soft-delete / retention. Without it a GCS keystore is rejected
  at load. S3 and Azure do not require this GCS-specific configuration flag:
  their startup probes detect version IDs and check current-object behavior.
  Those probes do **not** establish the absence of soft-delete, historical
  copies, backups, retention, or replication that preserves revoked keys.
  Operators must independently verify these policies for **every provider**;
  see [key durability and cryptographic deletion](disaster-recovery.md#key-durability-and-cryptographic-deletion).

## Storage class (data tiering)

Set `storage_class` on the **primary** backend to place new data uploads on a
cheaper cloud tier. It maps to the provider's per-object class header
(`x-amz-storage-class` on S3, `x-goog-storage-class` on GCS, `x-ms-access-tier`
on Azure) and is applied on every new data PUT.

**Allowed (synchronous-retrieval) classes — safe for the data path:**

| Provider | Allowed | Rejected (asynchronous retrieval) |
| --- | --- | --- |
| S3 | `STANDARD`, `STANDARD_IA`, `ONEZONE_IA`, `INTELLIGENT_TIERING`, `GLACIER_IR` | `GLACIER` (Flexible), `DEEP_ARCHIVE` |
| GCS | `STANDARD`, `NEARLINE`, `COLDLINE`, `ARCHIVE` | None of these |
| Azure | `Hot`, `Cool`, `Cold` | `Archive` |

All supported class names are matched case-insensitively and canonicalized;
Azure emits `Hot`/`Cool`/`Cold`. An unknown or empty value is rejected at config load.

**Why async classes are refused.** objectveil serves cold reads **synchronously**:
a `GET` on an object in an asynchronous-retrieval class returns
`InvalidObjectState` until a provider restore completes (minutes to ~12 hours),
which would hang or fail the read that triggered it. The loader therefore
refuses those classes on the data path rather than let a read stall in
production. `GLACIER_IR` (S3 Glacier **Instant** Retrieval) is allowed because
its reads are synchronous despite the "Glacier" name.

GCS `ARCHIVE` also supports synchronous, millisecond access. Its 365-day minimum
storage duration and retrieval charges are cost considerations, not an offline
restore requirement. See [Google storage classes](https://docs.cloud.google.com/storage/docs/storage-classes).

**Scope and caveats:**

- **New PUTs only.** The class affects objects uploaded after the setting takes
  effect. Existing objects keep their prior class — use a provider lifecycle
  policy to transition already-stored objects.
- **Primary only.** The `keystore` is excluded by design (crypto-shred keys must
  stay on the STANDARD tier so they remain instantly gettable and truly
  deletable); the `secondary` DR bucket is out of scope and keeps its
  bucket-default class. Setting `storage_class` on either is rejected at load.
- **S3 `INTELLIGENT_TIERING`:** allowed, but do **not** enable the bucket's
  optional *Archive Access* / *Deep Archive Access* tiers on an objectveil data
  bucket — those move aged objects into asynchronous retrieval, reintroducing
  the stalled-read hazard that config-time validation cannot see.
- Cross-region replication/copy may not preserve the source object's class;
  confirm your replication rules if you rely on the tier at the destination.

## Validation Rules

The daemon rejects the config file when any of the following are true:

- `format` is not exactly `objectveil/1`, or `namespace_id` is not a canonical lowercase non-nil UUID
- an unknown key is present (the loader rejects misspelled fields instead of ignoring them)
- `depot_path` does not exist
- `depot_path` is not a directory
- `state_dir` is a symbolic link, equals or contains `depot_path`, or overlaps an NFS backend root
- `max_cache_bytes == 0`
- `max_reported_file_bytes == 0`
- both `low_watermark_gb` and `low_watermark_bytes` are set
- the effective low watermark is below 1 MiB
- the effective low watermark exceeds the effective max cache size
- `upload_threads == 0` in write mode
- `restore_threads == 0`
- `restore_chunk_threads == 0` or `restore_chunk_threads > 64`
- `metrics_interval_secs == 0`
- `pack_target_chunk_bytes == 0`, `pack_flush_deadline_secs == 0` or `backfill_rate == 0`
- `pack_max_file_bytes == 0` or `pack_max_file_bytes` exceeds 1 GiB
- `compaction_live_fraction` is not a finite value in `(0.0, 1.0]`
- `access_batch_size == 0` or `access_batch_size > 1,000,000`
- `access_flush_interval_secs == 0`
- `access_max_pending_entries < access_batch_size` or `access_max_pending_entries > 1,000,000`
- `access_log_enabled = true` and both `access_es_url` and `access_pg_url` are set
- `access_log_enabled = true` (the default) and **neither** `access_es_url` nor `access_pg_url` is set — set `access_log_enabled = false` explicitly to disable access logging
- `access_pg_url` is set and `access_pg_table` is not a valid SQL identifier or schema-qualified identifier
- `access_pg_url` has no explicit `sslmode`, uses `disable`/`allow`/`prefer`, or uses `require` together with `access_pg_ca_cert`
- `access_es_url` is not an `http://`/`https://` URL with a host, or is `http://` while `access_es_auth` or URL credentials are set
- `access_es_url` is set and `access_es_index` is empty, starts with `_` or `.`, or contains characters other than ASCII alphanumerics, `-`, `_` and `.`
- `access_es_ca_cert` or `access_pg_ca_cert` is missing, unreadable or not a PEM certificate
- `access_failure_warn_interval_secs == 0`
- `access_failure_backoff_initial_secs == 0`
- `access_failure_backoff_max_secs == 0`
- `access_failure_backoff_initial_secs > access_failure_backoff_max_secs`
- `access_spool_dir` is set without `access_spool_max_bytes`
- `access_spool_max_bytes == 0`
- the access spool path contains a symbolic link or `..` component, or overlaps archives, other state, a local backend, a credential file or another configured output
- `log_rotate_max_bytes == 0`, or log rotation is configured without `log_file`
- `log_rotate_keep == 0` or `failed_uploads_log_rotate_keep == 0`
- a log, PID or metrics path breaks the [output destination](#output-destinations) rules
- `shim_socket_mode` exceeds `0o770` or grants any world access (`mode & 0o007 != 0`), or is a decimal value that looks like an octal mode typed by mistake
- `access_socket_mode` exceeds `0o770` or grants any world access (`mode & 0o007 != 0`), or is a decimal value that looks like an octal mode typed by mistake
- `primary` is a cloud backend (`azure`/`s3`/`gcs`) and no `keystore` block is configured (a cloud primary requires a separate split-bucket keystore; there is no single-bucket fallback)
- the `keystore` backend is `nfs` (the keystore must be a cloud object store that can do create-once)
- the `keystore` resolves to the **same bucket/container as `primary` or `secondary`** (split-bucket is mandatory: keys must live in their own non-versioned bucket, else a data-bucket's versioning/soft-delete preserves a shredded key)
- the `keystore` backend is `gcs` and `keystore.gcs_versioning_disabled_confirmed` is not `true` (the current backend interface cannot inspect GCS bucket policies; the operator must attest the keystore bucket has no versioning/soft-delete/retention)
- `storage_class` is set on a `keystore`, `secondary` or `nfs` backend, or names an unknown or asynchronous-retrieval class
- a backend sets `verify_ssl: false`, `ca_cert_path` or `server_side_encryption: true`, or sets `endpoint` without `endpoint_pin: true` (any `endpoint` on `gcs`)
- an Azure backend sets both `sas_token` and `account_key`; an S3 backend sets `credentials_file`, or only one of `access_key`/`secret_key`
- the selected backend is missing a required backend field (`bucket` for `s3`/`gcs`, `container` and `account_name` for `azure`, `path` for `nfs`)

## Runtime Behavior

- The shim server starts early, but FETCH, STAT, DELETE, ENSURE, and MODE return a retryable error until the full engine has catalog and revocation authority.
- The fanotify watcher starts only when `read_only = false`. It also reconciles the depot every five minutes after the preceding scan completes, so a closed temporary file renamed after an earlier scan is eventually uploaded. Failed recovery scans retain their retry request and retry after a 30-second cooldown without requiring another filesystem event. These scans inspect local files and hash tracked clean files, so large depots incur periodic disk I/O.
- The startup scan starts only when both of the following are true:
  - `read_only = false`
  - `skip_startup_scan = false`
- Every untracked local archive found by the writer startup scan is queued as Dirty for upload, whether the manifest is empty or populated. Existing Clean files are checked for size and content changes before eviction is enabled.
- The periodic stats logger starts only when `stats_interval_secs > 0`.
- The Prometheus exporter starts only when `metrics_file` is set.
- The access writer and access socket start only when both of the following are true:
  - `access_log_enabled = true`
  - exactly one of `access_es_url` or `access_pg_url` is set

## Shim Environment Variables

The preload shim is configured by environment variables, not by the daemon JSON.

Use only `OBJECTVEIL_*` and `OBJECTVEIL_SHIM_*` variables. The configured
root is mandatory; omitting `OBJECTVEIL_DEPOT` refuses intercepted operations.
For an intentionally unmanaged process, `OBJECTVEIL_SHIM_DISABLED=1` explicitly
disables interception. Do not use this setting on managed archive writers.
Socket pointers use the native versioned envelope. For custom `state_dir`,
set socket paths to the selected state's actual sockets.

| Variable | Required | Default | Notes |
| --- | --- | --- | --- |
| `OBJECTVEIL_DEPOT` | yes | none | Absolute depot root to intercept. |
| `OBJECTVEIL_SOCK` | no | `$OBJECTVEIL_DEPOT/.objectveil/shim.sock` | Override the shim FETCH socket path. |
| `OBJECTVEIL_ACCESS_SOCK` | no | `$OBJECTVEIL_DEPOT/.objectveil/access.sock` | Override the access datagram socket path. |
| `OBJECTVEIL_NEGATIVE_CACHE_TTL_SECS` | no | `60` | TTL for repeated `NOTFOUND` results; `0` disables the cache. Max accepted value is `3600`; a larger or unparseable value falls back to the default. |
| `OBJECTVEIL_MAX_FILE_BYTES` | no | `MAX_REPORTED_FILE_BYTES` (64 GiB) | Shim-side hard cap on daemon-reported file sizes (mirrors the daemon's `max_reported_file_bytes`); a larger reported size fails the operation as retryable. Accepted values must be `> 0` and `< u64::MAX / 2`; anything else falls back to the default. |
| `OBJECTVEIL_RPC_TIMEOUT_MS` | no | `5000` | Per-RPC send/recv timeout for shim → daemon FETCH/STAT/DELETE, in milliseconds. A parsed override is clamped to `[1000, 60000]` (an unparseable value falls back to the default). The `1000` floor is coordinated with — and compile-time-guaranteed to exceed — the daemon's inline serve-wait, so a low override can't spuriously fast-fail a cold read while the daemon is briefly holding the reply. |
| `OBJECTVEIL_RESTORE_MAX_WAIT_MS` | no | `300000` (5 min) | Absolute deadline for the full FETCH progress-poll exchange, including RPCs, polls and sleeps. Continued progress does not reset it; a healthy slow restore can exceed the deadline and the open fails. Each RPC is also bounded by its per-RPC timeout and remaining total time. A value outside `[1, 3600000]` (1 hour), or one that does not parse, falls back to the default rather than being clamped; the result is then raised to at least `OBJECTVEIL_RPC_TIMEOUT_MS`. Choose a budget appropriate to the workload and inspect failures before retrying. |
| `OBJECTVEIL_SHIM_DISABLED` | no | `0` | `1` disables interception for an intentionally unmanaged process; `0` or unset keeps it on. Any other value is a fatal configuration error. Never set it on a managed archive writer. |

## Prometheus Metrics

When `metrics_file` is set, the daemon writes the following metrics:

- `objectveil_uploads_total{result="success|failure"}`
- `objectveil_restores_total{result="success|failure"}`
- `objectveil_restores_polled_total` — cold restores slow enough to be served via the `RESTORING` progress-poll path rather than in one round-trip (large files, or a slow/degraded/throttled backend). Compare against `objectveil_restores_total{result="success"}` for the fraction of reads that were slow.
- `objectveil_deletes_total{result="success|failure"}`
- `objectveil_evictions_total`
- `objectveil_manifest_counter_drift_total`
- `objectveil_backend_requests_total{class="get|head|list|put|delete"}`
- `objectveil_backend_bytes_total{direction="downloaded|uploaded"}`
- `objectveil_secondary_fallbacks_total`
- `objectveil_files{state="dirty|uploading|clean|total"}`
- `objectveil_cache_bytes`
- `objectveil_cache_max_bytes`
- packing counters — `objectveil_files_packed_total`, `objectveil_chunk_puts_total`,
  `objectveil_chunk_gets_total`, `objectveil_files_served_from_chunk_total`,
  `objectveil_bytes_over_fetched_total`, `objectveil_per_file_fallbacks_total`,
  `objectveil_catalog_shard_fetches_total`, `objectveil_compactions_total`,
  `objectveil_chunks_reclaimed_total` and `objectveil_orphan_deletes_total`.
  Read them with the [cost model](cost-model.md#interpret-the-daemon-counters-correctly);
  the last two stay at zero while physical chunk cleanup is deferred.
- license gauges — `objectveil_license_info`, `objectveil_license_capacity_state`,
  `objectveil_license_managed_bytes`, `objectveil_license_soft_ceiling_bytes`,
  `objectveil_license_hard_ceiling_bytes`, `objectveil_license_expiry_state` and
  `objectveil_license_expires_at_unix`; see [licensing](LICENSING.md).
- `objectveil_shim_rejected_peer_total`
- `objectveil_shim_rejected_protocol_total`
- `objectveil_watcher_rejected_writer_total`
- `objectveil_watcher_auth_by_fd_owner_total`
- `objectveil_access_log_dgram_dropped_total`
- `objectveil_access_log_peer_rejected_total`
- `objectveil_access_log_truncated_total`
- `objectveil_access_spool_invalid_dropped_total`

For the `failure` label, the counters record terminal failures only. Uploads
increment the failure totals only after the daemon exhausts its retry budget
(initial attempt plus three retries) or proves the object is irrecoverably
missing. Backend-unavailable errors keep the file queued and do not increment
the Prometheus failure totals.

The hardening counters surface allowlist-rejection and writer-attribution
events that are also visible in the daemon log:

- `objectveil_shim_rejected_peer_total` counts shim-socket connections dropped
  because the peer UID was not on `shim_allowed_uids`.
- `objectveil_shim_rejected_protocol_total` counts shim connections refused for an unsupported or missing protocol prefix.
- `objectveil_watcher_rejected_writer_total` counts fanotify CLOSE_WRITE events
  dropped because the writer was not on `watcher_allowed_uids` or because the
  writer task vanished and the fd-owner fallback also failed.
- `objectveil_watcher_auth_by_fd_owner_total` counts events admitted via the
  fd-owner fallback when the writer task had already exited; non-zero rates
  on a busy commit server are expected.
- `objectveil_access_log_dgram_dropped_total` counts access-log datagrams that
  the receiver dropped because the bounded in-process queue was saturated
  (`access_max_pending_entries`). This drop happens *before* the on-disk
  spool, so it is the first place to look when access-log volume disappears
  under burst load.
- `objectveil_access_log_peer_rejected_total` counts datagrams dropped because
  the peer UID was not on `access_allowed_uids` or peer credentials were
  unavailable.
- `objectveil_access_log_truncated_total` counts access-log datagrams the kernel
  truncated because the sender's payload exceeded the receive buffer; a
  truncated datagram is dropped rather than logged partially. Non-zero values
  point at oversized access-log records and are delta-synced from the
  access-log layer.

## Access Logging

When access logging is enabled and a remote sink is configured:

- the shim emits newline-delimited depot-relative paths over `.objectveil/access.sock`
- the daemon deduplicates the batch by path
- the daemon rate-limits repeated access-log failure warnings
- the daemon retries failed remote writes with exponential backoff
- the daemon auto-disables the remote sink after `access_disable_after_failures` consecutive failures unless that setting is `0`
- the daemon writes:
  - Elasticsearch documents with `schema_version`, `path`, `last_access`, and `size`, or
  - PostgreSQL upserts into `access_pg_table` with columns `path`, `last_access`, and `size`

If `access_spool_max_bytes` is set:

- undeliverable batches are written to the access-log spool first
- spool files are replayed oldest-first before newer live batches
- the daemon exclusively locks an owned, mode-0700 spool directory; existing
  directories are never chmodded or adopted merely because filenames resemble
  spool batches. Unknown files, symlinks, hard links, and changed ownership are
  refused without deleting them.
- an older nonempty spool without the new ownership receipt is preserved and
  refused. Before upgrading, drain it with the old daemon where possible, or
  retain the directory and select a new empty private spool path. Do not copy or
  fabricate an ownership receipt to force adoption; retain old undelivered
  batches for explicit recovery.
- after an interrupted spool publication, restart the daemon to reconcile its
  own staging files before new batches can be accepted. The process lock remains
  held while any detached disk operation completes.

### Drop boundaries

Access-log loss can happen at two distinct points in the pipeline.
The spool only protects one of them:

1. **Receiver-side drop (pre-spool).** The shim posts datagrams into a
   bounded in-process queue sized by `access_max_pending_entries`.
   When the queue is full the receiver uses `try_send`, so the
   datagram is dropped immediately and `objectveil_access_log_dgram_dropped_total`
   is incremented. The on-disk spool sits *after* this queue and does
   not absorb pre-queue overflow. If access-log volume disappears
   during a burst and `access_log_dgram_dropped_total` is climbing,
   raise `access_max_pending_entries` (or fix the slow downstream sink).
2. **Sink-side failure (post-queue).** Once a batch is pulled off the
   queue it is handed to the configured Elasticsearch / PostgreSQL
   sink. If the sink call fails the controller backs off, retries,
   and (when `access_spool_max_bytes` is set; `access_spool_dir` only
   relocates the spool) spills the batch to disk so it can be replayed
   once the sink recovers.

If manifest reads fail persistently, the logger retains one bounded live batch
and stops draining the receiver until it can resolve the records' sizes. On
shutdown or receiver closure it makes a final attempt, then reports any
unsavable records as `unresolved_entries_dropped`; it never substitutes a
fabricated zero size. The spool cannot preserve records that could not yet be
resolved into a batch.

### Shim-side batching

The shim batches access events in a 32 KiB thread-local buffer and
flushes the buffer to the daemon socket via `sendto(2)` in any of
four cases:

- the next event would overflow the buffer
- the buffer reaches the ~30 KiB high-water mark
- a subsequent access event observes that the oldest buffered event is more
  than one second old (there is no independent per-thread timer)
- the thread exits (Drop on the thread-local buffer drains the
  remainder before the thread tears down)

An idle thread can retain buffered events longer than one second. Thread-exit
flushing is best effort; process death, failed datagram sends and socket/queue
pressure can lose events before they reach the daemon. The daemon's spool/retry
contract applies only after receipt and resolution of a batch. Access analytics
are not a lossless audit or file-durability mechanism.

The spool never grows past `access_spool_max_bytes`; when full, new undeliverable
batches are dropped.

If the remote sink is down and no spool is configured, undeliverable batches are dropped in memory after the bounded in-process queue and batch are drained.

For PostgreSQL:

- `access_pg_url` must include authentication and an explicit `sslmode` of
  `require`, `verify-ca` or `verify-full`
- `access_pg_ca_cert` supplies the PEM root CA when PostgreSQL TLS uses a private
  CA; pair it with `verify-ca` or `verify-full`
- the daemon creates `access_pg_table` (`path TEXT PRIMARY KEY`, `last_access
  BIGINT`, `size BIGINT`) if it does not already exist. It does not create a
  schema: for a schema-qualified name the schema must exist and the role must be
  allowed to create the table in it

## Path Safety

User-provided depot-relative paths are rejected when they are:

- empty
- absolute
- traversal paths such as `../escape`
- paths that enter the reserved `.objectveil` directory
