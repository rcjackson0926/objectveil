# Native causal catalog

The remote catalog is the authority for logical archive paths. A local redb
catalog is a recoverable mirror. Every native catalog envelope requires
`format: "objectveil-catalog"`, version `3`, and a checksum over the complete
canonical JSON payload under the `objectveil-catalog-v3` domain. Other versions,
missing labels and invalid checksums are refused. Checksums detect corruption;
they are not signatures against an actor able to replace both record and hash.

## Objects and ordering

A path selects a four-character lowercase hexadecimal shard from its packing
bucket. A writer publishes immutable objects under:

- `catalog/<prefix>/<era>.<daemon_id>.<seq>.delta`: batches of Put and Remove.
- `catalog/<prefix>/<era>.<daemon_id>.base.<seq>.idx`: merged snapshots.
- `catalog/epochs/`: permanent create-once writer-era tickets.

Era/order receipts establish causal precedence. Process identity and sequence
avoid same-name publication collisions. A replacement writer reserves a later
era before publishing; an old request arriving late cannot supersede its newer
authority. A failed request can leave a sequence gap. Clock time is not used to
infer which archive generation should win.

This is a fenced single-writer system. Tickets do not stop a second writer from
running, and checksummed merges are not a distributed lease or an active-active
write protocol. Every writer must use the same native generation. Never remove
or restore an earlier snapshot of the era-ticket history.

## Checked loading and folding

The reader inventories and merges complete shard history. It checks the object
size, envelope, checksum, exact embedded prefix, shard membership of every path,
base live/tombstone uniqueness and disjointness, and causal receipt invariants.
A malformed object, listing failure or continuation that cannot be completed
makes authority incomplete. It cannot authorize a restore, fold or deletion by
being treated as an empty shard.

A Put contains the full mapping: packed-member extent, immutable standalone
object, or standalone shared-blob mapping with a `dedup` record. A Remove is an
authoritative tombstone. Native dedup has no independent route publisher or
alternate route reader; a populated `routes/` namespace is refused.

A fold merges all relevant bases and deltas, writes its complete replacement
base durably, then retires only the exact inputs covered by its receipt. Causal
watermarks suppress already-folded records; they do not permit a partial listing
to discard unobserved history. Per-prefix serialization and immutable names keep
an in-process fold from racing its own publications.

## Crash and host-loss behavior

Payload and recovery-descriptor publication precede the corresponding catalog
Put. Local Clean follows durable catalog success for the captured generation.
A crash can leave extra immutable objects; their existence alone does not make a
logical path committed. Conversely, a successful remote Put followed by local
failure is recoverable by loading remote authority again.

The deletion intent stays durable until its Remove and required local cleanup
complete. An earlier deletion must not overwrite a newer re-add. The latest
Remove remains until a newer Put supersedes it; current code does not prune
tombstones by age or by a reference walk.

After local manifest loss, remote mappings can reconstruct current authority.
If conflicting historical generations survive without adequate authority, rebuild
refuses to guess. Shared blobs do not independently encode every referring path,
so losing their catalog as well as local state is not a supported reconstruction.

## Namespace boundary

A paired native storage marker validates the configured deployment UUID and role
before backend operations. The format marker is control metadata, not a catalog
shard. Internal key prefixes cannot alias depot paths. Native marker validation
does not verify all archive bytes or prove a converted dataset is complete;
conversion and cutover require independent verification.

See [packing](cold-read-chunk-packing.md), [deduplication](content-dedup.md),
[deletion epochs](obliterate-readd-epoch.md) and [recovery](../disaster-recovery.md).
