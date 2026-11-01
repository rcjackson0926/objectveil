# Cost model: when cold-read chunk packing can save

Packing can reduce data-object requests for a localized cold workload. It can
also increase transferred bytes and metadata work. Use this model to separate
those effects, then measure the complete workload. See
[packing operations](enabling-chunk-packing.md), the
[packing design](design/cold-read-chunk-packing.md), and
[verification cost](verify-cost-tiers.md).

The numerical rates below are hypothetical inputs, not current provider quotes,
ObjectVeil benchmarks or customer results. Substitute your actual provider,
region, class, retrieval, transfer and request charges before making a decision.

## What the runtime coalesces

Eligible small files are stored together in immutable `chunks/<hex>` objects.
Concurrent requests for members of the same chunk can share a whole-chunk
restore. That restore may also warm neighboring members whose current catalog,
key, content and local-publication checks succeed. It does not guarantee that
every footer member becomes a local file. An isolated small member may instead
use a bounded range read. Subsequent verified warm hits need no payload download.

These alternatives depend on layout, concurrency and available metadata. A
whole-chunk download can itself use several backend range GETs. Retries, HEADs,
namespace checks and key/catalog operations add requests. Coalescing therefore
does not promise one provider request per chunk, a fixed latency improvement or
freedom from throttling.

Read-only caches check remote authority on every new managed open, including
warm hits, and hash verified local contents. Catalog traffic is not merely a
one-time bootstrap cost. Writer-local access and read-only access have different
request and I/O profiles; measure the mode you will deploy.

## A simplified payload-only model

For an explicitly defined cold workload, let:

- `N` be payload GET requests in a measured or hypothetical per-file layout;
- `M` be payload GET requests in the packed layout, including ranges and retries;
- `B_ind` and `B_pack` be the respective transferred payload bytes in decimal GB;
- `g` be the assumed equal charge per payload GET;
- `e` be the assumed equal transfer charge per GB.

The payload-only comparison is:

```text
individual_payload_cost = N * g + B_ind * e
packed_payload_cost     = M * g + B_pack * e
payload_saving          = (N - M) * g - (B_pack - B_ind) * e
```

Compression, ciphertext framing and retrieval shape can change both byte totals.
For a simplified equal-encoding comparison where the only extra packed transfer
is unwanted neighboring content, write `B_over = B_pack - B_ind`. When `M > 0`
and `g > 0`, and `overfetch = B_over / M`, positive payload savings require:

```text
N / M > 1 + (e * overfetch) / g
```

With zero transfer charge, the break-even ratio is **1**, not zero: this model
saves request cost only when `N > M`. Equality means no saving on these two lines.
With billed over-fetch, more request reduction is needed. If request prices vary
by operation or tier, sum each actual request class instead of using one `g`.

This is not the total deployment cost. Add catalog and namespace LIST/GET/HEAD
requests, key operations, retries, retrieval charges, storage minimums, retained
historical objects, local storage, CPU/I/O, network, backup, operations and
licensing. A payload-only saving can coexist with a higher total bill.

## Illustrative scenarios

Assume `g = $0.0004 / 1000 = $0.0000004` per payload GET and
`e = $0.09 / GB`. These are arithmetic inputs only. Each scenario assumes its
stated GET count already includes payload ranges/retries; it makes no claim
about the count a particular ObjectVeil workload will achieve.

### Localized small-file workload

Suppose 50,000 files averaging 8 KiB transfer approximately 0.41 GB in either
layout, and the packed layout uses 40 payload GETs rather than 50,000:

| Payload component | Per-file layout | Packed layout |
| --- | ---: | ---: |
| GET requests | 50,000 | 40 |
| GET cost | $0.020000 | $0.000016 |
| Transferred payload | about 0.41 GB | about 0.41 GB |

The payload-request difference is **$0.019984 per workload run**; equal transfer
bytes cancel. At 500 identical runs per day, that arithmetic difference is about
$9.99 per day before excluded costs. It does not establish the total request
ratio, elapsed-time improvement or total savings. In particular, per-open
read-only authority checks still occur.

### Poor locality with billed over-fetch

Suppose a second workload uses 20,000 per-file payload GETs or 400 packed payload
GETs, with the packed layout transferring an additional 20 GB:

```text
request saving = (20,000 - 400) * $0.0000004 = $0.00784
extra transfer = 20 * $0.09                 = $1.80
payload saving = $0.00784 - $1.80           = -$1.79216
```

Here packing costs more on the payload lines. If transfer were actually free,
these same assumptions would save $0.00784 in payload request charges, without
establishing a latency or total-cost advantage. Lowering the chunk target or
standalone cutoff may reduce over-fetch; verify the effect with the same
workload and provider measurements.

## Interpret the daemon counters correctly

Take counter deltas over the same representative workload window. The following
names are stats fields; Prometheus uses the `objectveil_` prefix and `_total`
suffix.

| Counter | What it establishes | What it does not establish |
| --- | --- | --- |
| `files_served_from_chunk` | Requested cold reads served through a packed-member path, including range and coalesced cases. | It is not every prefetched member or every later verified warm hit, and is not a direct count of saved GETs. |
| `chunk_gets` | Whole-chunk download operations on the restore path. | A download may use multiple provider range requests; isolated member-range requests are not whole-chunk downloads. |
| `bytes_over_fetched` | Whole-chunk transfer beyond the specifically requested member, as accounted by the restore path. | Neighbors may later be useful; this is not an invoice or total network traffic counter. |
| `catalog_shard_fetches` | Nonempty shard walks for bootstrap, freshness and folding; a truncated nonempty read can also be counted. | One walk can read many objects and listing pages. It is not an object GET count or a mostly one-time cost. |
| `per_file_fallbacks` | Packed reads retried through a newly authoritative native standalone publication. | It is not the standalone share of the dataset or evidence that backfill would reduce its cost. |

Do not compute billed requests as `files_served_from_chunk - chunk_gets`, or
multiply `catalog_shard_fetches` by a GET price as if each walk were one request.
Use backend operation/byte counters as diagnostics and reconcile them with
provider request, retrieval and transfer metrics for the actual billed scope.
Namespace wrappers and SDK retries can perform underlying work not represented
one-for-one by an outer operation counter. Count successes and refusals alongside
latency so an incomplete workload cannot appear cheaper merely by doing less work.

## Tuning and retained storage

- Lower `pack_target_chunk_bytes` when whole-chunk over-fetch dominates; measure
  request count and throughput again. Target changes affect new publications.
- Lower `pack_max_file_bytes` to send more future files through immutable
  standalone storage. The in-memory packing capture ceiling also applies;
  see the configuration and packing guides. Existing objects are not rewritten
  simply because a cutoff changes.
- Native standalone payloads already have authoritative catalog mappings.
  Backfill excludes `standalone/` payloads: it is only for eligible, explicitly
  prepared path-addressed candidates inside a native namespace. A catalog miss
  does not fall back to an uncatalogued raw object, and backfill is not an
  incompatible-format converter.
- Compaction can improve layout but retains old chunk payloads, seals and recovery
  metadata. Physical chunk and unknown-blob sweeps are disabled pending a durable
  address-retirement protocol. Immutable standalone history can also remain.
  Model retained physical capacity rather than assuming current logical bytes
  equal billed remote storage. Known-content dedup GC has separate safety rules.
- Use immediately readable storage classes. Class minimum-duration/size charges,
  retrieval fees and asynchronous archival transitions can invalidate a model
  based only on nominal GB-month rates.

Evaluate cold and warm reads, writer publication backlog, read-only authority
traffic, tail latency and errors together. Packing changes resource tradeoffs;
it does not make uploads synchronously durable or replace the recovery plan.
