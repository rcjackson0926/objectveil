# Obliterate and re-add: per-path key epochs

Perforce can reuse a physical archive path after obliteration, and mutable RCS
archives can change in place. ObjectVeil therefore tracks content generations and
key epochs separately from pathnames. This is a single authoritative writer
protocol; writer-era catalog ordering is not distributed leader election.

## Durable identities

`PATH_EPOCH` records the local current epoch for a storage key and survives deletion
of its FILES row. Each captured file generation carries its epoch through Dirty
and Uploading. A key lookup resolves the exact remote poison chain to the first
live or unused slot. Only the exact native poison marker authorizes advancing;
malformed bytes and uncertain reads fail closed.

Epoch zero is `keys/<storage_key>`. Subsequent slots use the reserved
`keys/__objectveil_raw_hex__/epoch/<epoch>/<storage_key>` namespace. The raw-key
escape prevents an ordinary archive name from colliding with internal metadata.
A key slot is created atomically once. Revocation overwrites the slot with poison
rather than leaving a hole that a delayed create request could refill.

The local epoch is an ingestion guard, not the complete authority after a lost
manifest. A replacement writer must reconcile remote epoch history before mutation.
A restored file retains the epoch that authenticated its bytes. Missing live keys
or poison history require recovery assessment, not guessed reconstruction.

## Captured upload generation

An upload must still own its captured Uploading generation before key mint and
publication. The deletion read drain covers that decision. A local replacement or
obliteration cannot cause an old operation to retag a newer row or silently mint
under a new epoch. A still-current generation may reconcile a forward remote epoch;
a stale or removed generation cannot use that path to resurrect obsolete bytes.

Pending deletion blocks key mint and packed/standalone publication. The upload's
successful immutable payload and durable catalog Put precede its transition to
Clean. Errors preserve retryable local work; they do not prove remote completeness.

## Deletion transaction

The authoritative writer holds the exclusive deletion drain across cooperating
upload, restore, eviction and maintenance operations:

1. Reconcile remote key authority and persist a pending intent for the selected
   old epoch. Advance local epoch state before revoking that key.
2. Remove the corresponding local plaintext and synchronize the parent directory.
   Even an already-absent retry synchronizes the surviving parent. A provably
   newer local generation is retained.
3. Poison the intent's selected epoch. A retry keeps this identity instead of
   targeting whatever newer key happens to exist when the retry executes.
4. Remove the obsolete path-addressed primary payload, publish the durable catalog
   Remove and retire its local mapping. If complete authority establishes a newer
   already-published re-add, preserve that key and mapping.
5. Remove only the obsolete local FILES generation and clear the pending intent
   last. Startup resumes unfinished intents before admitting conflicting work.

A failed unlink, directory sync, key operation or catalog publication returns an
error and retains the intent. This protocol is not a cross-provider transaction;
its durable intent and idempotent retry sequence cover partial progress.

## Re-add and shared content

A genuine later write uses a fresh epoch/key. Identical plaintext does not justify
reusing a revoked path key. Shared-content deduplication also requires the path's
live key to match the content anchor. Where an existing path key cannot safely
share that anchor, publication uses an ordinary immutable standalone object.

Per-path revocation leaves a shared encrypted blob available to other authorized
paths. Destroying its content anchor requires complete last-reference proof;
unknown blobs remain retained. See [content deduplication](content-dedup.md).

## Deployment and recovery boundaries

Exactly one fenced writer may ingest or delete for a namespace; read-only peers
reject both operations. Local locks do not fence another host. Every application
writer must participate in the supported shim coordination before eviction is
safe. A deletion request identifies the current path, not an arbitrary historical
Perforce revision, so callers must serialize the intended application operation.

Remote revocation prevents new authorized reads; it does not erase another
machine's plaintext disk, already-open descriptors or copies outside the cache.
Retention of old key versions defeats crypto-shred, while loss of a live key
permanently loses encrypted archives. Unencrypted NFS data has no cryptographic
erasure guarantee.

Total writer loss can lose local writes not yet published remotely, even when
Perforce acknowledged them. The archive catalog is also not a backup of Perforce
metadata. See [crypto-shred](obliterate-crypto-shred.md),
[the causal catalog](catalog-shard-crdt.md) and
[disaster recovery](../disaster-recovery.md).
