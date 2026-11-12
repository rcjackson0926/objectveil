# Enabling content de-duplication

Content de-duplication stores identical large archive content once in cold
storage even when it appears at several depot paths. Small packed files are
not deduplicated. Perforce already lazy-copies branches, so measure the
remaining opportunity before enabling this feature.

The feature is off by default. Enabling it changes deletion semantics:
**obliterate revokes a path first; destruction of the shared content key waits
for background garbage collection, even for content with only one referrer.**
A surviving referrer keeps the shared bytes decryptable.

## Measure first

Stop the daemon or use a consistent manifest snapshot. The measurement tool
opens the manifest read-only and cannot share the live daemon's database lock.

```bash
objectveil-dedup-measure /path/to/config.json --manifest /snapshots/manifest.redb
objectveil-dedup-measure /path/to/config.json --manifest /snapshots/manifest.redb --json
```

Use the **standalone** scope to assess opportunity. The duplicate count is
exact for the selected records; reclaimable byte totals are logical estimates,
not a measurement of physical cloud bills. Packed and combined figures are
informational. The tool is included in every release package and bundle.

## Preconditions

1. Exactly one writer ingests or obliterates against this backend. Read-only
   caches are allowed. The attestation below does not fence a second writer.
2. Use a separate keystore with versioning, soft delete, retention, and
   key-preserving replication disabled. Startup probes detect enabled object
   versioning on S3 and Azure only; GCS requires an explicit operator
   attestation. No probe can establish that soft delete, retention, backups or
   replication are absent, so verify those policies yourself on every provider.
3. All daemons and shims have been upgraded together, p4d has been restarted,
   and old writer binaries are fenced. Preserve remote catalog history,
   writer-era tickets, data, and the current
   key/poison history needed for recovery.
4. The deletion semantics are acceptable for this data, and the measured
   opportunity justifies additional metadata and operational costs.

## Enable

Set these fields on the single writer and restart it:

```jsonc
{
  "dedup_enabled": true,
  "dedup_gc_single_writer_ack": true
}
```

`dedup_enabled` enables new dedup uploads. The second field explicitly attests
that this is the sole writer and allows known-content Tier-2 GC after complete
remote-authority checks. Without the attestation, shared keys are retained and
a warning is logged. Unknown orphan blobs are conservatively retained even
with the attestation; their deletion cannot yet be proved safe.

The local cache still evicts durably uploaded objects, including on the commit
server. Active or unuploaded files remain local. Existing non-dedup objects
remain readable and are not rewritten merely by enabling the flag.

## Resulting behavior

- New eligible standalone content uses a random content key and immutable
  `blobs/<ciphertext hash>` object. Identical content may reuse that object.
- The authoritative path mapping lives in the checksummed causal catalog.
  No route-only or raw-path fallback bypasses that authority.
- If a mutable path's existing serving key differs from the shared content
  key, its update uses ordinary immutable standalone storage. Deduplication
  savings can therefore be lower for rewritten archive paths.
- Read-only caches check remote freshness on every new archive open. An
  existing open descriptor keeps its old snapshot. Writer local reads and
  automatic eviction remain enabled.
- Obliterate removes the path's authority and poisons its serving key. Tier-2
  GC destroys a shared key only after complete remote metadata and local
  coordination prove that no live path needs the content.

## Garbage-collection cost and timing

Tier-2 GC runs on the writer. Every five minutes it looks in its local manifest
for content that no path references any more. When it finds some, it proves the
whole batch (up to 4,096 records) at once:

1. it reads the complete remote catalog once, to confirm that no path still
   names the content or its blob;
2. it reads every object in the keystore once, to confirm that no other key
   slot holds the same key; and
3. for each record that passes both, it downloads and decrypts the shared blob
   to authenticate it under its content key.

Uploads, restores and obliterates keep running during all three. They wait only
while a proved record's key is actually destroyed, which takes one small
keystore read and one small write per record.

The cost of steps 1 and 2 follows the size of the deployment, not the amount
being collected: every catalog object and every key object is read once per
pass, however many records are in the batch. To keep that a small share of the
time, the collector rests ten times as long as a pass took before it runs those
reads again, so on a large deployment passes can be hours apart. An obliterate
shortens that rest to the length of the last pass and re-examines every
retained record, so destruction follows a deletion within about one pass.

A record that cannot be collected is retained and retried with a delay that
doubles from ten minutes up to one day:

- **Still referenced.** Another path's catalog entry names the content, or
  another key slot holds the same key. The usual cause is a rewritten mutable
  path that kept its original key; the shared key and blob are then retained
  until that path is obliterated. The collector remembers which object blocked
  the record and re-reads only that object on each retry.
- **Not provable.** The catalog or keystore could not be read completely, the
  content key is not live, the blob does not authenticate or could not be
  downloaded within one hour, or the key write failed. Each logs a
  `content-GC` warning.

Every pass that has something to prove ends with a `content-GC pass finished`
log line that counts the records it destroyed, retained as still referenced,
and failed to prove. Retry delays and remembered blockers are kept in memory
only: after a restart the first pass proves every unreferenced record again.

## Disable or recover

Set `dedup_enabled: false` and restart to stop new dedup uploads. Existing
shared objects stay readable through their durable mappings; disabling the flag
also parks shared-content GC. This does not convert old dedup objects back to
independent copies.

Use the integrated binaries for this configuration rollback. **Do not roll
back only the binaries after new catalog metadata has been published.** Older
writers or readers cannot safely interpret the new authority.

A replacement machine can recover from the complete remote catalog and keys.
Shared blob names cannot reconstruct referring paths if both that catalog and
the local manifest are lost. Preserve catalog metadata as carefully as payloads.
See [disaster recovery](disaster-recovery.md).

Deduplication does not make submissions synchronously durable. Losing the
writer and its disks before asynchronous upload completes can still lose newly
acknowledged files. Zero-loss protection requires independent synchronous
durability or replication.

## Deletion and confidentiality limits

A path deletion does not destroy content still referenced elsewhere. Someone
with raw access to the shared blob and a retained key can still decrypt it
until the final reference is removed and Tier-2 GC runs. Retained key versions,
backups, local plaintext, and already-open files further limit any deletion
guarantee. Leave dedup disabled if every path deletion must immediately destroy
its independent encryption key without waiting for shared-content GC.

Hashes and shared-object metadata expose equality and allow guesses about
low-entropy content to be tested. The content key itself is random, not derived
from those public hashes. The complete key and publication design is in
[content de-duplication](design/content-dedup.md).
