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
| `upload_threads` | 8 | **16–32** (match cloud SDK connection-pool limit) |
| `restore_threads` | 16 | 4 (cold reads are rare here) |
| `restore_chunk_threshold_mb` | 8 | 8 (no change) |
| `restore_chunk_threads` | 8 | 4 |
| `max_cache_gb` | 100 | fit 7 days of submit volume |
| `low_watermark_gb` | 80 | 0.8× `max_cache_gb` |
| `eviction_target` | derived | derived |

Rationale: the hot path is upload. Extra upload threads amortize
cloud-SDK request overhead. Restore budget is reduced because read
traffic is thin here; give the CPU/bandwidth back to uploads.

### B. Read-heavy edge replicas

Characteristic: steady `p4 sync` load from many clients; commits are
rare. The cache runs cold almost all the time.

| knob | default | read-heavy |
|---|---|---|
| `upload_threads` | 8 | 2 (edge rarely writes) |
| `restore_threads` | 16 | 16 for read-only mode; its separate validation admission limit is fixed at 16. Test 24–32 only for writer-mode restore workers. |
| `restore_chunk_threshold_mb` | 8 | 8 (default already ranges large blobs; lower only for an all-large-file workload) |
| `restore_chunk_threads` | 8 | 16 |
| `max_cache_gb` | 100 | as large as your disk allows — hit-rate scales with working-set fit |
| `low_watermark_gb` | 80 | 0.9× `max_cache_gb` (less eviction churn) |

Measure read-only authority requests and full-file hashing alongside payload
throughput. Read-only validation admits at most 16 operations, including
unfinished blocking hashes, independently of `restore_threads`; overload is
retryable. Increasing worker configuration does not remove that limit or a
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
| `objectveil_files{state="uploading"}` | in-flight uploads | should oscillate around `upload_threads`; if pinned at ceiling, upload workers are saturated |
| `objectveil_evictions_total` | files evicted | rate of growth = churn; sustained high rate = working-set overrun |
| `objectveil_manifest_counter_drift_total` | drift between atomic counters and redb truth | should be 0; any increment = a bug, file an issue |

## Startup tuning

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
