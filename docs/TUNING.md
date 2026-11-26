# Performance tuning guide

The built-in defaults target a mid-sized shop with bursty writes
and mostly cold reads. Real deployments benefit from tuning a handful
of knobs against actual workload shape.

## Workload shapes

Pick the profile closest to your deployment, then use the suggested
starting values below. These are starting points — measure, don't
guess.

### A. Write-heavy commit servers

Characteristic: many concurrent `p4 submit` connections, high rate of
`FAN_CLOSE_WRITE` events, hot uploads saturating the upstream link.

| knob | default | write-heavy |
|---|---|---|
| `upload_threads` | 8 | Start at 8, then trial 16 if the host and backend have headroom |
| `restore_threads` | 16 | 4 (cold reads are rare here) |
| `restore_chunk_threshold_mb` | 8 | 8 (no change) |
| `restore_chunk_threads` | 8 | 4 |
| `max_cache_gb` | 100 | fit 7 days of submit volume |
| `low_watermark_gb` | 80 | 0.8× `max_cache_gb` |
| `eviction_target` | derived | derived |

`upload_threads` admits up to four concurrent archives per configured unit.
Settings of 8 and 16 allow up to 32 and 64 captures or payload transfers
respectively. Each active multipart object can hold four 8 MiB parts, so
those settings can hold up to 1 GiB and 2 GiB of multipart payloads
respectively, plus SDK, packer and staging overhead. Small archives can wait
for a pack to seal before going clean.

Leave `upload_publication_overlap` at its default of `false` for live parallel
submits. For a cache catching up on files already on disk, setting it to
`true` lets a standalone payload release its transfer slot once its bytes
are durable. At the settings above, up to eight additional tasks can be
scheduled while descriptors and catalog entries finish publishing. An archive
becomes clean only after its catalog update succeeds. Restart the daemon to
apply a change. In a live Azure comparison using the same 432 staged files,
overlap raised the startup-to-durable rate from 129.9 to 152.9 MiB/s. With
three Perforce clients submitting concurrently, the corresponding rates were
146.0 MiB/s with overlap disabled and 105.9 MiB/s with overlap enabled.

Standalone payload transfers can run concurrently within a catalog shard,
while the shard serializes catalog publication and compaction to preserve
revision order.
Concurrent puts waiting on one shard can share a durable delta containing
up to 32 entries. A writer with an idle shard publishes immediately. Deletes,
compaction and catalog snapshots close the current group.
With the legacy catalog, archive paths with the same leading directory share a shard.
A trial with a separate top-level directory per file can therefore overstate
publication throughput for a depot that stores millions of revisions under one
top-level directory. Preserve the directory layout in a representative trial.

During a burst, queued object operations can share a check of the configured
storage identity markers. A plaintext depot has one primary marker.
For packed archives, upload workers compress members concurrently and encrypt
them when a keystore is configured.
A sealed chunk publishes its catalog entries before the cache commits the
matching local generations as clean in one manifest transaction.
For a partitioned catalog, all packs share up to four concurrent partition
updates per configured upload thread, with a minimum of eight and a maximum
of 64. Reads of nonempty partition history are limited separately. Packs
can start sealing at their own deadlines while another seal is in progress.
At an estimated 128 catalog objects in a partition, the running cache can
acknowledge a durable update and let its compactor merge the old records in
the background. At 256 objects it checks the actual count and requires room
before admitting another update. Finite migration jobs fold inline.

To profile a slow drain, set
`RUST_LOG=info,objectveil::publication_profile=debug` for the daemon during
a representative submit. The log records elapsed milliseconds for lock
acquisition, preparation, payload upload, descriptor or seal-marker upload,
catalog publication and local completion. Standalone records include archive
paths. Add `objectveil::engine::catalog_shard=debug` to the filter for
per-partition predecessor lookup and delta PUT timings.

Measure on the production storage path using representative archive sizes,
multiple concurrent Perforce clients and the parallel submit settings you
use in production. Compare archive bytes written per second with durable
archive progress and watch the Dirty/Uploading counts.
A single manifest snapshot supplies the file-state gauges at each metrics interval,
so an archive moving between states counts once in the reported total.
A queue that grows throughout repeated workload intervals means the cache
is falling behind. `objectveil_backend_bytes_total{direction="uploaded"}`
measures stored bytes sent to the backend, which can differ from archive
bytes because of compression, deduplication, framing and metadata traffic.
Compare multiple concurrency settings with the same workload. Watch CPU,
disk IO, cloud throttling and the time to drain the queue after the final
submit. Allow capacity for bursts and recovery scans even when the
sustained rate keeps up.

Track the change in `objectveil_backend_requests_total{class="get"}` per
completed archive as well as MiB/s. A workload with many small archives can
spend most of its time on metadata requests. When a fresh cloud listing and
an earlier catalog download report the same strong ETag, the engine can
reuse that catalog record from a bounded in-memory cache. Backends without
a strong ETag fetch the record again.

AzCopy can tune request concurrency while benchmarking a storage endpoint.
rclone separates concurrent file transfers from the number of parts in flight
per file. An adaptive ObjectVeil uploader would need per-request latency and
retry measurements and a shared byte budget across files and parts. The
existing logical PUT counter counts a whole multipart object once, so it
cannot by itself detect a saturated part uploader. The current 10,000-part
limit caps one stored object at 78.125 GiB, including encryption framing.
rclone increases the part size when a known object size would exceed the part
limit. ObjectVeil could do the same if it also reduced concurrent parts when
larger buffers would exceed the memory budget. [AzCopy performance tuning](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-optimize) · [rclone multipart uploads](https://rclone.org/s3/#multipart-uploads)

A live Azure comparison with the legacy catalog used three parallel Perforce
clients and four rounds
of submits, producing 72 incompressible 16 MiB revisions and 72 companion
maps under one leading directory. With eight upload threads, the cache drained
all 144 archives at 78.5 MiB/s from the first submit. The 24-job migration
published and cold-verified them at 20.0 MiB/s. Spreading the revisions across
six leading directories gave 73.5 MiB/s for the cache and 22.4 MiB/s for
migration. On the one-directory corpus and the same endpoint, rclone with
24 transfers copied at 117.4 MiB/s. Following the copy
with a full download check at 16 checkers gave a combined 50.1 MiB/s.
The encrypted ObjectVeil configuration in that trial also published per-path keys and a recoverable catalog.
Record copy time, verified completion time, and request counts separately when
repeating the comparison on a production link. [rclone download checks](https://rclone.org/commands/rclone_check/)

### Planning for a very large depot

Size the catalog from the number and distribution of *physical archive
paths*. In particular, Perforce companion map files have their own entries.
The legacy layout limits one catalog base to 32 MiB and a normal shard walk
to 128 MiB. Once compaction cannot produce a bounded base, deltas accumulate
and a reader can eventually refuse an incomplete inventory. For scale,
400 decimal TB in 16 MiB revisions is about 23.8 million paths before map
files. At 399 bytes per entry in a synthetic sample, a single directory could
require about 8.9 GiB of legacy base metadata.

Set `"catalog_layout": "partitioned"` when initializing a new large installation.
The full canonical archive key selects one of 65,536 partitions, including
for archives that share a pack. Payloads keep their existing addresses and
encryption keys. A compacted partition has a checksummed root. A small root
holds its records inline. Larger roots reference immutable leaves with
ordered key ranges and content digests. A single archive lookup reads only the
intersecting leaf and outstanding changes in its partition. An incomplete
inventory or missing referenced leaf stops operations that require those records.
The design uses immutable metadata and publishes references after their contents.
[lakeFS internals](https://docs.lakefs.io/concepts/internals/) · [restic write ordering](https://restic.readthedocs.io/en/stable/100_references.html#read-and-write-ordering)

A synthetic test placed 23,841,858 revision paths under one leading directory,
published each partition with the production encoder, and read back every
descriptor. All 65,536 partitions were populated. The largest held 444 paths.
This represents the catalog for 400 decimal TB at 16 MiB per revision, before
companion map files. A separate inventory test walked 1,000,001 descriptors in
104 seconds with 37 MiB of peak process memory. Both tests exercised metadata
with synthetic storage. Actual payload throughput depends on the storage link
and the mix of archives.

A cloud configuration without a keystore defaults to the partitioned layout.
Encrypted cloud and NFS configurations default to legacy. Preserve the layout
of any existing namespace. For a conversion,
upgrade all clients and stop or fence daemons, readers, migration jobs, and
maintenance tools that access the namespace.
Run `objectveil-rebuild-catalog <config.json> --upgrade-catalog` on the writer
host. Keep those processes stopped throughout retries. The tool builds a private
on-disk index in the state directory, stages the new catalog alongside the old
one, and reads back every pointer and deletion record. Provision space on the
state volume for that working index. After verification it switches the storage
format marker. Set `catalog_layout` to `partitioned` on every client before
restarting. The old format marker cannot be restored as a rollback once the new
catalog has accepted writes. Conversion transfers metadata only. The original
migration job can resume or run `--verify-only` with the updated configuration.
An existing migration-only installation can run the converter with its own
configuration and state directory. Its source directory does not need an
archive-root claim. The converter checks any claim already present.

Compaction can leave superseded immutable catalog segments in storage so
in-flight readers can finish. To reclaim them, fence every reader and writer
again, then run
`objectveil-rebuild-catalog <config.json> --prune-catalog-segments` from an
initialized writer installation. Keep the fence until the command exits.
Pruning verifies the catalog roots before deleting unreferenced segments.

By default, migration starts cold verification of a committed archive while other
archives upload. Each freed publisher slot can start another archive. For
workloads that prioritize getting copies
into Blob Storage earlier, `--upload-first` publishes a bounded window of
archives before reading that window back on the same host. Count and spool
limits force verification to catch up. Both modes report
the time of the last observed archive publication (`last_publication_seconds`)
and `verification_tail_seconds`, time spent draining remaining verification work after publication. Measure
both final publication and fully verified completion on the Perforce host.

Two trials of each mode on a four-core OpenStack test host used a live Azure
Blob endpoint, 24 jobs, 72 incompressible 16 MiB revisions and 72 companion
maps. The earlier batch-based migrator published the last archive in 33–37 seconds
with `--upload-first` and 47–82 seconds with the default. Full cold verification
finished in 49–53 and 49–83 seconds respectively. The reverse-order pair
finished verification faster with the default. All 72 large blobs in every
trial had committed multipart block lists. In a separate parallel-submit test
on the same host, the partitioned cache drained 144 archives in 18.0 seconds
from the first submit (64.1 MiB/s). The submits themselves finished in 7.9
seconds. All 144 archives subsequently restored byte-for-byte.

With the rolling scheduler, a same-source encrypted Azure comparison using
24 jobs published all 144 files in 21.1 seconds and finished the process,
including independent cold verification, in 25.2 seconds. The earlier scheduler
took 45.2 and 46.9 seconds respectively on that source. A separate
`--upload-first` trial encountered a 30-second transport timeout. Keep provider
retries and local CPU load with the timing results when comparing runs.

On a snapshot-capable filesystem, a future incremental recovery mode could
enumerate changed paths between two retained snapshots and hash just those
archives. The checkpoint would advance after every change in the interval
was accounted for, including renames and hard links. A gap in the snapshot
history would trigger full reconciliation. ZFS exposes file changes with
`zfs diff`, and Btrfs supports incremental send from a read-only parent
snapshot. [OpenZFS snapshot differences](https://openzfs.github.io/openzfs-docs/man/master/8/zfs-diff.8.html) · [Btrfs incremental send](https://btrfs.readthedocs.io/en/latest/btrfs-send.html)

### B. Read-heavy edge replicas

Characteristic: steady `p4 sync` load from many clients; commits are
rare. The cache runs cold almost all the time.

| knob | default | read-heavy |
|---|---|---|
| `upload_threads` | 8 | 2 (edge rarely writes) |
| `restore_threads` | 16 | 16 for read-only mode; its separate validation pool has 16 workers and admits up to 64 ENSURE jobs. Test 24–32 only for writer-mode restore workers. |
| `restore_chunk_threshold_mb` | 8 | 8 (default already ranges large blobs; lower only for an all-large-file workload) |
| `restore_chunk_threads` | 8 | 16 |
| `max_cache_gb` | 100 | as large as your disk allows — hit-rate scales with working-set fit |
| `low_watermark_gb` | 80 | 0.9× `max_cache_gb` (less eviction churn) |

Measure read-only authority requests and full-file hashing alongside payload
throughput. Read-only validation has 16 active workers and admits up to 64 ENSURE jobs,
including queued work and unfinished blocking hashes, independently of
`restore_threads`. Admission overflow returns BUSY; the matching shim waits
within its operation deadline. Increasing worker configuration does not remove that limit or a
shim's absolute restore deadline. Range downloads can matter above
`restore_chunk_threshold_mb` (default 8 MiB), but backend latency and bandwidth
must be measured on the actual deployment.

### C. Balanced (the default shape)

Leave defaults. Monitor the `cache_bytes / max_cache_bytes` percentage
in the periodic stats logger; if it sits at 95–100% continuously and
the eviction counter climbs fast, either raise `max_cache_gb` or
lower `low_watermark_gb` (i.e. drop more aggressively).

## Eviction watermarks

Eviction fires when `cache_bytes` rises above the low watermark; the
engine then drops least-recently-accessed files one at a time until
`cache_bytes` falls to or below the derived `eviction_target`
(`max(1, low_watermark * 90%)`). `eviction_target` is a byte floor,
not configurable and not a per-wake-up file count.

- **Thrashing symptom**: evict counter climbs monotonically, cache %
  oscillates narrowly near `max_cache_gb`. Cause: working set is
  larger than `max_cache_gb - low_watermark_gb` divided by
  the churn window. Fix: either raise `max_cache_gb` or lower
  `low_watermark_gb` so each eviction pass reclaims more.
- **Stall symptom**: cache hits `max_cache_gb`, uploads start stalling
  because local disk fills. Fix: lower `low_watermark_gb` so eviction
  reclaims before saturation.

## Cloud-specific notes

- **Cloud (S3 / Azure / GCS)**: all three go through one `object_store`
  wrapper, so `restore_chunk_threshold_mb` + `restore_chunk_threads`
  drive parallel range GET uniformly — there is no per-cloud SDK with
  its own range behaviour. A file at or above `restore_chunk_threshold_mb`
  is fetched with `restore_chunk_threads` parallel range GETs
  (`restore_chunk_threads > 1`); set the threshold to `0` to disable
  ranging entirely. If you see low per-file restore throughput for large
  blobs, lower `restore_chunk_threshold_mb` so ranging kicks in sooner.
- **NFS**: `restore_chunk_*` knobs are no-ops (the NFS backend reports
  no range-read capability). NFS reads go through the kernel page
  cache. The useful knobs are `max_cache_gb` and `low_watermark_gb`
  only.

## Storage class (cost tiering)

Set `storage_class` on the **primary** backend to land new data uploads on a
colder, cheaper tier when your access pattern justifies the per-GB savings
against the tier's retrieval/minimum-duration costs. Applies to **new PUTs
only** — existing objects keep their prior class (transition them with a
provider lifecycle policy).

- **Safe (synchronous-retrieval) classes** — the only ones accepted on the data
  path, because objectveil serves cold reads synchronously:
  - **S3**: `STANDARD`, `STANDARD_IA`, `ONEZONE_IA`, `INTELLIGENT_TIERING`, `GLACIER_IR`
  - **GCS**: `STANDARD`, `NEARLINE`, `COLDLINE`, `ARCHIVE`
  - **Azure**: `Hot`, `Cool`, `Cold`
- **Rejected at config load** — asynchronous-retrieval classes (S3 `GLACIER`
  Flexible + `DEEP_ARCHIVE`, Azure `Archive`). A `GET` on such an
  object returns `InvalidObjectState` until a minutes-to-12h restore finishes,
  which would stall the synchronous cold read that triggered it. Use `GLACIER_IR`
  (S3 Glacier **Instant** Retrieval) if you want a Glacier-priced tier that still
  reads synchronously.
- A good default for cost savings on cold-but-occasionally-read history is
  `STANDARD_IA` (S3) / `NEARLINE` (GCS) / `Cool` (Azure). Only the **primary**
  data backend honours the setting — the keystore and secondary are excluded.
- S3 `INTELLIGENT_TIERING` is allowed, but do **not** turn on the bucket's
  optional Archive/Deep-Archive access tiers for an objectveil data bucket: aged
  objects would move to asynchronous retrieval and cold reads of them would fail.

See [`CONFIGURATION.md`](CONFIGURATION.md#storage-class-data-tiering) for the
full rules and rationale.

## Access-log sinks

- **Elasticsearch**: each remote flush carries up to `access_batch_size`
  deduplicated events (default 10000); tune that plus `access_es_url` +
  TLS settings and `access_flush_interval_secs`.
- **Postgres**: each flush upserts up to `access_batch_size` rows
  (default 10000). The upsert resolves conflicts on the `path` primary key
  the daemon creates with the table, so no extra index is needed; if you
  see lag, check that key exists on a pre-created table and that
  autovacuum is keeping up.
- **Spool fallback**: `access_spool_max_bytes` bounds the on-disk buffer
  during sink outages. Size it to ride through a peak-hour outage. It is
  unset (disabled) by default; leave it unset to disable the spool (a
  value of `0` is rejected at config load).

## Metrics to watch

Prometheus textfile (path set by `metrics_file`):

| metric | meaning | threshold |
|---|---|---|
| `objectveil_cache_bytes / objectveil_cache_max_bytes` | fill % | > 95% sustained → raise cap or lower watermark |
| `objectveil_uploads_total{result="failure"}` | terminal upload failures | any persistent growth → check the failed-uploads log |
| `objectveil_restores_total{result="failure"}` | cold-read failures | any persistent growth → backend health |
| `objectveil_files{state="uploading"}` | archives being processed or awaiting pack publication | compare with the active archive task limit of `4 × upload_threads` and with the dirty queue trend |
| `objectveil_evictions_total` | files evicted | rate of growth = churn; sustained high rate = working-set overrun |
| `objectveil_manifest_counter_drift_total` | drift between atomic counters and redb truth | should be 0; any increment = a bug, file an issue |

## Startup tuning

When Perforce closes a staging file before renaming it into place, the Linux
watcher follows its open file descriptor for up to five seconds. It can track
256 such files at once. If the name remains unresolved or the queue fills, the
watcher schedules a depot recovery scan. Recovery discovers untracked files
in one directory walk, then checks local manifest rows in pages of 256.
It still hashes resident clean archives to detect same-size writes. The daemon
logs `running scheduled depot recovery reconciliation` when it starts a
scheduled pass.

- Leave `skip_startup_scan = false` on writers unless independent controls prove
  there were no unrecorded archive changes. The scan queues untracked files and
  marks changed Clean files Dirty before enabling eviction. Skipping it for
  performance alone can leave writes unpublished and manifest metadata stale;
  the final eviction checks do not replace that publication work.
- `stats_interval_secs = 0` disables the periodic stats line. Use on
  write-heavy hosts where log volume matters.
- `metrics_interval_secs = 5` for hosts scraped aggressively; default
  15s is fine for normal deployments.

## Known thin spots

- The eviction policy is plain LRU. Frequency-weighted or
  size-weighted would help read-heavy workloads with a long tail of
  cold files. Planned but not implemented.
- No per-path priority (e.g., hot depots). All files share one LRU.
