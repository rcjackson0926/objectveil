# Cost model: when cold-read chunk packing can save

A practical model for estimating the **cloud-storage cost saving** that
chunk packing delivers when a cold `p4 sync` pulls many small archive
files back from an `object_store` backend. For *what packing is and how
it works* (packing is always on — there is no on/off switch), see
[`enabling-chunk-packing.md`](enabling-chunk-packing.md) and the design
[`design/cold-read-chunk-packing.md`](design/cold-read-chunk-packing.md).
For the **cost of offline-verifying** those archives at three integrity
depths (and the same `g`/`e` symbols reused), see
[`verify-cost-tiers.md`](verify-cost-tiers.md).

> **Rates below are illustrative inputs retained from the earlier model, not
> verified current provider quotes or ObjectVeil customer results.** Use them
> to reason about formulas, not to predict an invoice. Replace every rate with
> your provider contract, region, storage class and actual retrieval/transfer
> terms before quoting savings. Model all metadata/key requests as well as data
> requests. Savings and latency improvements are conditional, not guaranteed.

## Why packing can reduce data-object requests

Cloud object stores bill a cold read on **two independent lines**:

- **Per-request (GET):** a flat charge per object fetched, regardless of
  size.
- **Per-GB egress:** the bytes that leave the store (often the bigger,
  more variable line — and frequently **free or near-free** for
  in-region / same-cloud reads).

For a working set of **many small files**, the request line dominates
and, more importantly, the *per-file overhead* — a TLS handshake, a
round-trip, connection setup, provider rate-limiting — is the real pain,
not the bytes. Ten thousand 8 KiB GETs cost almost nothing in egress but
are ten thousand round-trips and ten thousand chances to hit a 503.

Packing collapses those many small per-file objects into a few large
`chunks/<hex>` objects. A suitably localized cold sync of **N** files can use fewer whole-chunk data
GETs than separate per-file payloads: the daemon coalesces
concurrent restores of co-packed files into a single whole-chunk fetch
(single-flight, keyed by chunk key) and warms *every* member of that
chunk into the local cache. So the saving is dominated by
**request-count reduction**, plus two large secondary wins:

- Fewer data-object round-trips can reduce overhead for a localized burst.
  Metadata/key checks, compression, coalescing waits and over-fetch still cost work.
- Lower data-object request rates can reduce throttling pressure; they do not
  prevent 503s, rate limits or other provider failures.

The catch is **over-fetch**: a whole-chunk GET pulls co-packed neighbours
that the current sync may not have asked for. When sync locality is good
those neighbours are exactly what the client wants next, so it is
*prefetch, not waste*. When locality is poor it is extra egress you paid
for and evicted unused — see [Caveats](#caveats).

## Historical illustrative pricing inputs

Ballpark **list** prices for a standard/hot tier. Read the warnings
above them; egress especially is tiered and contract-dependent.

| Provider (standard/hot tier) | GET request (~per 1,000) | Internet egress (~per GB) | Same-region / same-cloud read |
|---|---|---|---|
| AWS S3            | ~$0.0004 | ~$0.09 (tiers down at PB scale) | often free to in-region EC2 |
| Azure Blob (Hot)  | ~$0.0004 | ~$0.087 | often free in-region |
| Google Cloud Storage | ~$0.0004 | ~$0.12 | often free in-region |

Two things to internalise before using these:

- **The GET line is ~$0.0004 per 1,000 (~$0.40 per million) across all
  three.** That is the number the request-count reduction acts on.
- **Egress is the swingy line.** If your daemon reads from a bucket in
  the same region/cloud as the daemon, egress may be **$0/GB**, in which
  case over-fetch is free and packing's money win is essentially the
  full request-count reduction. If you egress to the public internet,
  over-fetch has a real price and can erode (or invert) the saving.

Symbols used below:

- `g` = cost per GET (dollars) — e.g. `$0.0004 / 1000 = $4e-7`.
- `e` = cost per GB egress (dollars) — e.g. `$0.09` on S3 internet, `$0`
  in-region.

## The formulas

Let **N** = files restored, **M** = distinct chunks actually fetched
(the coalesced GETs), `B_ind` = Σ(requested file bytes, GB), `B_over` =
over-fetched bytes (GB, the co-packed neighbours pulled but not
requested).

```
individual_cost = N × g   +   B_ind × e
packed_cost     = M × g   +   (B_ind + B_over) × e
saving          = individual_cost − packed_cost
                = (N − M) × g   −   B_over × e
```

The whole model is that one line: **you save one GET for every file that
now rides a shared chunk, and you pay egress on whatever you over-fetch.**

### Break-even

Packing saves money when `(N − M)·g > B_over·e`. Dividing by M and
writing `overfetch = B_over / M` (average over-fetched GB per chunk GET),
packing wins on the dollar lines once the coalescing ratio clears,
roughly:

```
N / M   >   (e × overfetch) / g
```

Read it as: **the more you over-fetch per chunk and the more egress
costs, the higher the coalescing ratio you need to come out ahead.**

- **Free/in-region egress (`e ≈ 0`)** → break-even `N/M ≈ 0`: packing
  wins on money essentially always, and the request/latency/throttling
  wins are pure gravy.
- **Billed internet egress + good locality** (`overfetch` per chunk near
  0) → break-even is tiny, packing wins easily.
- **Billed internet egress + poor locality** (each chunk mostly
  neighbours you didn't want) → break-even `N/M` can be in the
  thousands. Below it, packing costs money on the egress line even while
  it still slashes request count and latency. This is the case to tune
  for.

## Worked examples

### Example 1 — 50,000 files averaging 8 KiB, good locality (S3 list price)

A CI fleet cold-syncs a source tree: **N = 50,000**, mean 8 KiB, total
≈ **390 MiB (~0.41 GB)**. These files were submitted together, so they
co-pack well.

| | Individual | Packed |
|---|---|---|
| Cloud GETs | 50,000 | ~40 chunk GETs + ~10 catalog-shard fetches ≈ **50** |
| Bytes egressed | ~0.41 GB | ~0.41 GB (over-fetch ≈ 0 — the working set fills the chunks) |

The ~40 comes from the 390 MiB working set landing in a few dozen 64 MiB
chunk buckets (packing buckets by depot subtree); single-flight folds
every concurrent miss for a chunk into one GET.

- **Request line:** individual `50,000 × $4e-7 = $0.020`; packed
  `50 × $4e-7 ≈ $0.00002`. Request saving ≈ **$0.020 per cold sync**.
- **Egress line:** near-identical (~0.41 GB) either way, so ~$0 delta;
  `$0` entirely if reads are in-region.
- **Ratio: ~1,000× fewer requests** (50,000 → ~50).

The absolute dollar saving on one sync is small — but that fleet running
the sync 500×/day saves ~$10/day *in request charges alone*, does it with
**1,000× fewer round-trips** (much faster cold sync), and never comes
near S3's ~5,500-GET/s-per-prefix ceiling that 50,000 parallel GETs would
have flirted with (503 throttling). The latency and throttling wins are
the headline; the dollars are a bonus.

### Example 2 — 20,000 files averaging 512 KiB, poor locality (S3 internet egress)

A scattered restore: **N = 20,000**, mean 512 KiB, total ≈ **10 GiB
(~10.7 GB)**, but the files are spread across the depot and submitted at
different times, so each 64 MiB chunk holds only a little of what this
sync wants. Say the sync touches files living across **M = 400** chunks,
and on average only ~25% of each fetched chunk is wanted (over-fetch ≈ 48
MiB/chunk → `B_over ≈ 400 × 48 MiB ≈ 18.75 GiB ≈ 20 GB`).

- **Request line:** `(20,000 − 400) × $4e-7 ≈ $0.0078` saved.
- **Egress line (internet, `e = $0.09`):** over-fetch costs
  `20 GB × $0.09 = $1.80` *extra*.
- **Net (billed internet egress):** `$0.0078 − $1.80 ≈ −$1.79` — packing
  **loses money on the byte line** here. Check against break-even:
  `N/M = 50`, but `(e × overfetch)/g = (0.09 × 0.047)/4e-7 ≈ 10,500`. `50
  ≪ 10,500`, so the model correctly predicts a loss.
- **Net (in-region egress, `e = $0`):** over-fetch is free → packing is a
  clean win: 50× fewer requests, still faster, still no throttling.

The lesson: **poor locality + large-ish files + billed egress is exactly
where packing's money win shrinks or inverts.** The fix is tuning, not
turning packing off — lower `pack_target_chunk_bytes` (less over-fetch
per chunk GET) and/or lower `pack_max_file_bytes` so these mid-size files
stay standalone. See [Caveats](#caveats).

## Measure YOUR actual saving from the daemon's metrics

Do not trust the worked examples for your shop — measure. The daemon
exports the packing counters below (Prometheus textfile at `metrics_file`,
suffixed `_total` and prefixed `objectveil_`; the same values print in the
periodic stats line under the field names used here). Take two snapshots
bracketing a representative window and diff them.

| Metric (stats field / `objectveil_*_total`) | Meaning |
|---|---|
| `files_served_from_chunk` | Cold reads served *out of* a chunk (range fast-path or whole-chunk warm). Each of these would have been its own per-file GET before packing — the numerator of the win. |
| `chunk_gets` | Whole-chunk downloads actually issued on the restore path — this is **M**, the coalesced GETs. |
| `per_file_fallbacks` | Cold reads that refreshed a packed mapping and then used a current catalog-selected standalone payload. It does not authorize reads of uncatalogued raw-path objects. |
| `bytes_over_fetched` | Bytes pulled beyond the specifically-requested blob when a whole-chunk restore pulled the whole chunk to serve one file — the byte-line cost to subtract. |
| `catalog_shard_fetches` | Cloud catalog-shard objects fetched (cold-start / miss bootstrap + shard-compaction reads) — a small mostly-fixed GET overhead to net out. |

Over a window (`Δ` = end − start of each counter):

```
requests_saved  =  Δfiles_served_from_chunk  −  Δchunk_gets

dollars_saved   ≈  requests_saved            ×  g
                −  (Δbytes_over_fetched / 1 GB) × e        # over-fetch egress
                −  Δcatalog_shard_fetches     ×  g        # bootstrap GET overhead
```

The first term is the request-count win: files that rode a shared chunk
instead of each costing a GET. The two subtractions are packing's own
overheads — the egress you paid on over-fetch, and the handful of GETs
the catalog bootstrap adds. If `e ≈ 0` (in-region reads), drop the
over-fetch term and `dollars_saved ≈ requests_saved × g`.

Sanity checks while reading the window:

- `files_served_from_chunk / chunk_gets` is your **effective coalescing
  ratio** — the real N/M. Watch it trend up as more of the depot packs.
- `bytes_over_fetched` large relative to bytes actually served → poor
  locality; expect the egress subtraction to bite (see Caveats).
- A high `per_file_fallbacks` share means much of your working set is not
  packed yet — run backfill (Stage 2 in
  [`enabling-chunk-packing.md`](enabling-chunk-packing.md)) before
  judging the saving.

## Caveats

Be honest with these before quoting a saving:

- **Over-fetch is only free when locality is good.** The whole-chunk GET
  pulls co-packed neighbours; that is prefetch only if the sync wants
  them soon. Poor locality + billed egress can make packing *cost* money
  on the byte line even while it cuts requests — Example 2. Watch
  `bytes_over_fetched` and mind the break-even.
- **`pack_target_chunk_bytes` too large amplifies over-fetch.** Bigger
  chunks mean fewer requests but more bytes dragged along per isolated
  read. If `bytes_over_fetched` is high, lower it (default 64 MiB) or
  confirm your packing buckets align with sync locality.
- **`pack_max_file_bytes` bounds the blast radius.** Files at/above it
  (default 8 MiB) stay standalone and never over-fetch. If a class of
  mid-size files has poor locality, keeping them standalone (a lower
  cutoff) can beat packing them.
- **The catalog-shard bootstrap adds a few LIST/GETs on a cold start.**
  `catalog_shard_fetches` is small and mostly one-time per cold daemon,
  but net it out for an honest per-window figure — it is real cloud
  traffic.
- **Savings assume the files are actually co-packed.** Submit locality is
  what puts related files in the same chunk. Files scattered across many
  submits/subtrees pack poorly; that is a workload property, not a bug —
  tune `pack_target_chunk_bytes` / `pack_max_file_bytes`, or accept that
  this corner of the depot resolves via `per_file_fallbacks`.
- **In-region egress is often free — confirm before pricing bytes.** If
  it is, the egress terms vanish and the request-count reduction is the
  whole story (and packing effectively always wins).
