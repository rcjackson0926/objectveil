# Verification depth and estimated cloud cost

`objectveil-verify` checks whether archive revisions referenced by a Perforce
checkpoint can be recovered from the configured native storage namespace.
Choose presence, sample or full integrity checking explicitly. None of these
checks establishes that a live checkpoint includes submissions made after it
was captured, or replaces a Perforce metadata backup.

## Authority and scope

Each logical archive must resolve through the complete current native catalog.
Its descriptor selects an immutable standalone payload or an exact packed
member, including expected sizes, hashes and encryption information. An object
at the logical archive path cannot substitute for a missing catalog descriptor.
Deep verification obtains its expected identity from the remote catalog and
does not require the cache's local manifest.

The content pass groups references by physical backing object. A shared chunk
can therefore be downloaded once for several referenced members. Catalog and
namespace checks, the initial existence scan, retries, key reads and an optional
`p4 verify` pass can make additional requests. The displayed estimate is a
simplified accounting model, not a complete provider invoice.

## Tiers

| Tier | CLI | Content work | Guarantee and limitation |
|---|---|---|---|
| Presence | `--integrity presence` or `--tier 1` | HEAD the selected backing objects | Detects missing backing objects and standalone stored-size differences from native catalog descriptors. Packed payload/member integrity and same-size corruption require content reads. |
| Sample | `--integrity sample --sample-pct P --sample-round R` or `--tier 2` | Presence plus a deterministic slice of content reads | Checks selected objects and referenced members cryptographically. Unselected content is not verified in that round. |
| Full | `--integrity full` or `--tier 3` | Presence plus all selected content reads | Checks the distinct backing objects and referenced members against native catalog sizes, hashes and encryption authority. Reports missing, corrupt or unavailable dependencies as failures. |

Without `--integrity`, the command performs the existence scan and any requested
Perforce verification, without the content integrity pass. Use an explicit tier
for a scheduled integrity job.

Sampling uses stable hash bands of the physical object key. At `P` percent,
rounds `0` through `ceil(100/P)-1` cover every band of a fixed work set. Object
counts and bytes in a round need not equal exactly `P` percent. Coverage across
rounds is meaningful only for the same object generations; changing live data
requires a new verification scope. For example, ten consecutive 10-percent
rounds cover a fixed set, while repeatedly running round zero repeats its slice.

## Cost model

Let `N` be the distinct standalone payload count, `M` the distinct chunk count,
`S` their unique stored bytes in decimal GB, `g` the price per GET/HEAD, and `e`
the transfer price per GB. Let `Q` and `B` be the actual sampled object count and
sampled stored GB. The incremental backing-object model is:

```
Presence: (N + M) * g
Sample:   Presence + Q * g + B * e
Full:     Presence + (N + M) * g + S * e
```

These formulas exclude catalog traffic and other authority/identity requests,
key operations, the initial scan, retries, storage-tier retrieval fees and
provider-specific charges. Even presence checking reads metadata and catalog
bytes. Native namespace guards can perform underlying operations that are not
represented individually by the verifier's outer request counters. Confirm
actual charges using provider metrics for a representative workload.

For illustration only, at `g = $0.0004 / 1000`, `e = $0.09 / GB`, 40,000 chunks
and 2,000 GB give approximately $0.016 of backing-object presence requests and
$180 of full-pass content transfer, plus request and other overhead. With
100,000 standalone payloads totaling 0.8 GB, the same simplified full-pass
model gives $0.152 before overhead. These are historical hypothetical inputs,
not current quotes or promised workload costs. In-region transfer is not
universally free; use the actual provider, region, storage tier and contract.

Supply applicable model inputs with `--price-per-1k-requests` and
`--price-per-gb-egress`. Request and transfer savings depend on the layout and
workload; comparing with `p4 verify` requires measuring the same scope and
provider charges.

## Estimates, output and restarts

```
objectveil-verify config.json --checkpoint ckp.gz --estimate --sample-pct 10
objectveil-verify config.json --checkpoint ckp.gz --integrity full
```

`--estimate` resolves the work set and probes backing-object sizes without
fetching archive content. Catalog and namespace metadata reads still occur.
It projects the three tiers using the selected sample round and supplied prices;
it does not certify content integrity.

A real integrity run reports `head_requests`, `catalog_shard_gets`,
`content_gets`, `bytes_egressed`, `objects_hashed`, `hash_mismatches`,
`size_mismatches` and `missing_objects`, with an approximate cost. Treat missing
catalog authority, unavailable keys, unresolved objects, hash differences and
size differences as failed or incomplete verification, and inspect the log and
exit status before accepting a recovery set. A successful run describes the
observations made during that run, not a guarantee against later changes or
loss of another dependency.

Estimates acquire the verifier job lock before opening the progress log. They
refuse an active job using the same state directory, preserve saved scan state
even with `--reset-state`, and may create ownership-control files and a log.
Give independent jobs separate state directories and log paths. See
[verification jobs and restarts](OPERATIONS.md#verification-jobs-and-restarts)
and the [disaster recovery guide](disaster-recovery.md).
