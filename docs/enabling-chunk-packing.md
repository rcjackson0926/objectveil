# Cold-read chunk packing — operating notes

> **Packing is always on.** It was previously an opt-in `packing_enabled`
> flag; that toggle has been removed (a config that still names it loads
> with a deprecation warning and the field is ignored). Packing itself can
> no longer be staged on or rolled back; the staged rollout below applies
> only to the still-opt-in **backfill** of pre-existing standalone objects.

For *what it is and how it works*, see the design:
[`design/cold-read-chunk-packing.md`](design/cold-read-chunk-packing.md).
For the full config reference, see the packing rows in
[`CONFIGURATION.md`](CONFIGURATION.md).

Packing coalesces the many small per-revision archive objects into larger
content-addressed `chunks/<hex>` objects with a sharded catalog, so a
cold `p4 sync` of N files that miss the cache becomes **M cloud GETs with
M ≪ N** instead of one GET per file. What packs is governed by
`pack_max_file_bytes`: a file below it packs; a file at/above it stays a
standalone object (bounding over-fetch). There is no on/off switch — to
keep more files standalone, lower `pack_max_file_bytes`.

## Preconditions

- A matching Linux release with published validation evidence and verified
  artifact checksums. Packing is always on; validate the deployment and its
  recovery procedure before relying on eviction.
- A backend that can enumerate its object space (`object_store` cloud
  backends, or NFS), required for complete catalog authority and backfill.

## The knobs

| Knob | Default | Role |
|------|---------|------|
| `pack_target_chunk_bytes` | 64 MiB | Target encoded chunk size; the 64 MiB per-bucket memory reservation bound can seal earlier. |
| `pack_max_file_bytes` | 8 MiB | Files at/above the effective threshold stay standalone; in-memory capture has a 16 MiB ceiling. |
| `pack_flush_deadline_secs` | 30 | Seal a partially-filled chunk after this long so trickle submits land durably. |
| `compaction_live_fraction` | 0.5 | Compact a chunk once its live-byte fraction drops below this. |
| `backfill_enabled` | `false` | Opt-in repacker for pre-existing standalone objects. |
| `backfill_rate` | 50 | Backfill pacing, objects/second. |
| `backfill_prefix` | unset | Restrict the backfill walk to one depot-path subtree. |

The process admits at most 64 open/sealing L0 buckets and 256 MiB of conservative
packer reservations. At saturation, durable Dirty work backs off and retries;
archive bytes are not discarded. Streaming standalone, multipart and restore
buffers have separate bounds, so this is not a total-process memory limit.

## Recommended backfill rollout

Packing is always on and needs no rollout. What you *can* stage is
**backfill** — the opt-in migration of pre-existing standalone objects
into chunks (`backfill_enabled`, default off). Roll it out in stages and
watch the metrics between each. Every packing/backfill knob is
restart-required — it takes effect on daemon (re)start, not on hot reload
(see `CONFIGURATION.md`).

### Stage 0 — baseline (before backfill)

Confirm the gate is green and record a baseline: cloud GETs per synced
file, restore latency, and cloud request bill for a representative cold
sync. This is what backfilling the legacy objects has to beat.

### Stage 1 — new writes only (the default)

```jsonc
"backfill_enabled": false
```

New small submits pack into chunks (they always do); **pre-existing
objects stay standalone** and resolve through the catalog-miss →
per-key-GET fallback, so nothing old changes. This is the default state
with backfill off — it exercises the write/seal path, the catalog, and
single-flight whole-chunk restore on freshly-packed data without touching
anything already in the store.

Watch for: chunks being written, packed files being served from chunks,
and zero increase in restore failures. Let it run under real submit +
sync load before proceeding.

### Stage 2 — backfill legacy objects

```jsonc
"backfill_enabled": true,
"backfill_rate": 50,
"backfill_prefix": "depot/pilot/subtree"   // start scoped
```

The backfill loop walks existing standalone objects, repacks the small
ones into chunks, and **deletes the standalone copy only after the
catalog (and its cloud shard) resolves the key to the chunk**
(object-before-pointer). Start with a scoped `backfill_prefix` and a low
`backfill_rate`; confirm the migrated subtree still reads correctly and
that cold-sync GET counts drop, then widen the prefix and raise the rate.
It is resumable (a persisted cursor) and rate-limited, and safe to run
while serving.

### Stage 3 — steady state

Compaction (`compaction_live_fraction`) repacks live members and shard folding
consolidates catalog deltas automatically. Superseded chunks, seal markers and
recovery metadata remain stored; physical chunk and unknown-blob sweeps are
disabled until durable address retirement can prevent a delayed DELETE from
removing a later publication at the same address. Account for retained remote
bytes when planning capacity. Local cache eviction continues normally.

## What to watch

The daemon exposes packing counters alongside its other metrics:

- **`files_served_from_chunk` vs per-file fallbacks** — the coalescing
  ratio. Rising chunk-served share on cold syncs is the win.
- **`chunk_gets` / cloud GETs per synced file** — the headline number;
  target ≪ 1.
- **`bytes_over_fetched`** — whole-chunk over-fetch. If high relative to
  bytes served, lower `pack_target_chunk_bytes` or check that packing
  buckets align with your sync locality.
- **`compactions`** — completed spatial repacks. `chunks_reclaimed` and
  `orphan_deletes` remain zero while physical chunk cleanup is deferred; they
  must not be interpreted as remote storage savings.
- **`catalog_shard_fetches`** — cold-start catalog bootstrap traffic.

## Tuning

- **Chunk size** (`pack_target_chunk_bytes`): larger ⇒ fewer requests but
  more over-fetch per isolated read; smaller ⇒ the reverse. 64 MiB is a
  reasonable start.
- **Standalone cutoff** (`pack_max_file_bytes`): files above this are
  never packed. Keep it well below the chunk target.
- **Flush deadline**: shorter ⇒ lower write-durability latency for trickle
  submits, at the cost of smaller (less-coalesced) chunks.

## No on/off switch

There is no rollback flag: packing is always on. If you need most files to
land standalone (the old "packing off" behaviour), lower
`pack_max_file_bytes` so files exceed the standalone-bypass cutoff — the
write side then stores them as standalone objects, while any bytes already
in a chunk stay readable from that chunk (the read-side catalog resolve is
not gated on anything). Note this does NOT retire existing chunks; to fully
remove them you must re-materialize standalone objects from them first.
