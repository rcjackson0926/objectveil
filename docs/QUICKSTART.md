# Quick Start

Upgrading an existing installation? Follow the
[ObjectVeil upgrade and format guide](upgrading.md) before changing
services, state directories, environment variables or cloud prefixes.


Start with `INSTALL.txt`, at the top of the release bundle or in
`/usr/share/doc/objectveil/` for a package. It covers checksum verification,
installation, service ownership and coordinated first activation or upgrade.

Get `objectveil` running in front of an existing Perforce server, from a
verified release to a read-only cache to full write mode. This is the
narrative walkthrough; [`OPERATIONS.md`](OPERATIONS.md) has the terser
reference version and every operational detail (metrics, log rotation,
disaster recovery, obliterate, upgrade notes). [`CONFIGURATION.md`](CONFIGURATION.md)
is the full field-by-field config reference.

## Before you start

- A Linux host (fanotify + `LD_PRELOAD` are Linux-only) that can reach both
  your `p4d` server's depot filesystem and your chosen cloud backend.
- An existing `p4d` you can restart (the shim is loaded via `LD_PRELOAD`
  when `p4d` starts — no `p4d` patches, no protocol changes).
- **Two** cloud buckets/containers if you're using S3/Azure/GCS: one for
  the archive data, one **separate** one for the crypto-shred keystore.
  The split is mandatory (the daemon refuses to start without it) — see
  [the note below](#2-provision-storage). Skip this if you're doing an
  NFS-only proof of concept.
- The complete matching Linux release artifacts. No license is bundled: use
  a vendor-signed version 2 license for anything beyond a short trial. Without
  one the daemon starts on its built-in developer license (1 TiB, expiring 30
  days after the release was built); see [`LICENSING.md`](LICENSING.md).

## 1. Install the matching release

Follow `INSTALL.txt` to verify and install the Debian package, the RHEL 9 RPM
or a tar bundle in a new versioned directory. Use all nine runtime artifacts
from that same release: the daemon, initializer, migration tool, verifier,
manifest summary, catalog rebuild, rescan, deduplication measurement and
preload library. Do not mix a new daemon with an already mapped older shim.

The commands below use installed executable names. For a tar installation,
substitute the absolute paths under that verified bundle's `bin/` directory.

## 2. Provision storage

Pick one of [`docs/examples`](examples) as your starting point:

| Backend | Template |
|---|---|
| AWS S3 | [`objectveil.s3.example.jsonc`](examples/objectveil.s3.example.jsonc) |
| Azure Blob Storage | [`objectveil.azure.example.jsonc`](examples/objectveil.azure.example.jsonc) |
| Google Cloud Storage | [`objectveil.gcs.example.jsonc`](examples/objectveil.gcs.example.jsonc) |
| NFS only (no cloud) | [`objectveil.nfs.postgres.example.jsonc`](examples/objectveil.nfs.postgres.example.jsonc) |

Each cloud template already has the recommended thread/packing settings
filled in and active (see [`TUNING.md`](TUNING.md) and
[`design/cold-read-chunk-packing.md`](design/cold-read-chunk-packing.md) for
the reasoning) — the only things you need to change are marked
`REPLACE-ME`: `depot_path`, bucket/container names, region, and
credentials.

**Before you fill in the config**, provision the cloud side:

1. Create the **data** bucket/container (versioning/replication policy is
   up to you — this one benefits from durability features).
2. Create a **separate** **keystore** bucket/container for the per-file
   crypto-shred keys. This one must have **versioning OFF, soft-delete
   OFF, object-lock/WORM OFF, and no replication** — `p4 obliterate`
   destroys a key by overwriting it, and that actually destroys the revoked
   key only when no historical copy survives. Startup probes check
   visible storage behavior and exposed version IDs; they do not prove that
   soft delete, retention, backups or replication are absent. Independently
   verify those policies on every provider. GCS also requires the explicit
   `gcs_versioning_disabled_confirmed: true` attestation. Details:
   [`design/object-storage-best-practices.md`](design/object-storage-best-practices.md).
3. A secondary is an explicitly owned native ObjectVeil backend, not an
   arbitrary plaintext archive tree. It must have a matching native primary-role
   marker and catalog/key dependencies. Import existing local archives through
   `objectveil-migrate` instead of pointing a fallback at an unconverted tree.
   Secondary configuration alone does not establish a redundant copy.

Copy the template, fill in the `REPLACE-ME` values, and drop the
`.jsonc`-only comments if you prefer a plain `.json` file (the loader
doesn't parse `//` comments — the shipped `.jsonc` files are references to
copy from, not something you point the daemon at unmodified).

## 3. Initialize the local archive root

Set `format` to `objectveil/1` and replace the nil `namespace_id` template value
with a generated UUID. Preserve that UUID in every configuration sharing this
remote deployment. Under the archive-owner account, run:

```sh
objectveil-initialize /path/to/your-config.json
```

For an existing archive-only tree, first review its contents and then add
`--existing-archives`. This attestation does not establish a consistent snapshot.
Do not initialize database/journal roots or incompatible populated cache state.
See [upgrading](upgrading.md) for format transitions.

## 4. First run: read-only mode

Use a separate reader cache against an already populated backend. A read-only
cache never initializes storage, so a brand-new deployment has nothing to read
yet: start its first writer with [step 6](#6-go-to-write-mode), and return to
this drill once archives have been published. With `read_only: true`, the daemon serves restores and checks remote freshness even
for warm archive opens; managed writes are rejected. This confirms backend and
credentials without enabling fanotify or uploads. Do not apply read-only mode
to an active submit server:

```bash
/usr/bin/objectveil /path/to/your-config.json
```

Watch the startup log line (`objectveil starting`) and the periodic stats
line (`stats_interval_secs`, default 60s). If `metrics_file` is set, check
the Prometheus textfile output — `objectveil_restores_total` should climb as
`p4d` (once wired up in the next step) requests cold files.

If the daemon refuses to start, the error is almost always one of:
missing/wrong keystore bucket (see step 2), a backend field validation
error (see [`CONFIGURATION.md`](CONFIGURATION.md#validation-rules)), or a
`depot_path` that doesn't exist yet — create it first.

## 5. Wire up `p4d`

Restart `p4d` with the shim preloaded:

```bash
LD_PRELOAD=/path/to/libobjectveil_shim.so \
OBJECTVEIL_DEPOT=/p4/1/depots \
p4d ...
```

`OBJECTVEIL_DEPOT` must match `depot_path` in the daemon's config exactly.
The two socket-path env vars (`OBJECTVEIL_SOCK`, `OBJECTVEIL_ACCESS_SOCK`) are
optional — only set them if you moved the daemon's state directory away
from the default `depot_path/.objectveil`.

Run a `p4 sync` of something not already on local disk and confirm it
succeeds (silently, from `p4d`'s point of view — nothing about the sync
command changes). Check `objectveil_restores_total{result="success"}` ticked
up, or watch the daemon log for restore lines.

## 6. Go to write mode

Once cold restores are confirmed working, stop and externally fence p4d,
every archive writer and the cache, including automatic restarts. Hold the
fence through configuration and role changes. This is a coordinated cutover,
not a daemon-only toggle:

1. With all writers and the daemon stopped, set `read_only: false` in the config.
2. Grant the daemon `CAP_SYS_ADMIN` (needed for fanotify, which watches
   for new/changed depot files). The packaged systemd unit already does this
   for the service alone; for a manual launch:
   ```bash
   sudo setcap cap_sys_admin+ep /usr/bin/objectveil
   ```
3. Start the daemon, then restart every p4d with the matching current shim and
   actual root/socket paths.
   Release the fence only once every writer is coordinated. The shim caches its
   role for the process lifetime, so restarting only the daemon is insufficient.
   The daemon
   now runs the startup scan (adopting existing
   local files into the manifest), starts the fanotify watcher, and
   uploads new submits in the background.

Writer caches still evict safely uploaded archives. All writer processes must
use supported shim-coordinated writes. Upload is asynchronous: losing the writer
and its disks before remote upload completes can lose acknowledged submits.
Read [the recovery and coordinated upgrade requirements](disaster-recovery.md)
before relying on this as part of a backup or disaster-recovery plan.

The daemon drops `CAP_SYS_ADMIN` again immediately after fanotify
initializes (unless `keep_caps: true`), so it isn't running privileged
for its whole lifetime.

## 7. Production hardening (optional, but recommended)

- **Run under `systemd`** instead of the daemon's own `daemon: true` flag
  — use the supplied package or adapt its unit for the verified tar bundle. See
  [`OPERATIONS.md`](OPERATIONS.md#packages).
- **License:** obtain a signed version 2 license for the deployment and pass
  it via `--license <path>` (or the other search-path locations —
  [`LICENSING.md`](LICENSING.md)).
- **Metrics:** point Prometheus (or `node_exporter`'s textfile collector)
  at `metrics_file`.
- **Access-log analytics:** if you want per-path access history, set
  `access_log_enabled: true` and configure `access_es_url` or
  `access_pg_url` — see [`CONFIGURATION.md`](CONFIGURATION.md#access-logging).
  The cloud templates ship with this off so they load with no external
  dependency; the NFS + Postgres template and the full reference show a
  configured sink.
- **Tune for your workload:** the shipped thread/packing settings are a
  balanced default. If you're write-heavy (commit servers) or read-heavy
  (edge replicas), see the workload-shape tables in
  [`TUNING.md`](TUNING.md).

## What next

- [`OPERATIONS.md`](OPERATIONS.md) — full deployment reference: metrics,
  log rotation, checkpoint verification, the cache-integrity rescan tool,
  obliterate, and the shim/daemon upgrade-coupling notes.
- [`disaster-recovery.md`](disaster-recovery.md) — what to do if the
  local state is lost, including recovery dependencies and preservation of
  any surviving unpublished local files.
- [`ARCHITECTURE.md`](ARCHITECTURE.md) — how the pieces fit together.
