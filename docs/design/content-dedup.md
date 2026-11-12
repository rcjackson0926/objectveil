# Content de-duplication for cold storage

Deduplication is optional and disabled by default. This document describes
native ObjectVeil storage. Release qualification is established by the exact
build's validation evidence, not by this design document.

## Scope and key hierarchy

Deduplication applies to large standalone archive objects. Small files remain
packed members with their own offsets and accounting. Perforce already
lazy-copies branches; the opportunity here is identical bytes stored under
distinct archive paths.

| Object or record | Purpose |
| --- | --- |
| Per-path key slot, qualified by path epoch | Authorizes serving that path; obliterate poisons the slot. |
| `keys/.../by-content/{H}`, qualified by content epoch | Stores a random shared content key for plaintext hash `H`. |
| `blobs/{BLAKE3(ciphertext)}` | Immutable encrypted data shared by paths with identical content and key. |
| Checksummed causal catalog Put | Authoritative path-to-object mapping, including `CatalogEntry.dedup` and the complete `DedupRoute`. |
| Local `DEDUP_ROUTE`, `CONTENT_REFS`, and `CONTENT_BLOB` | Routing mirrors, reverse references, and known-content reclamation candidates; these are not sufficient proof for deletion by themselves. |

The shared key is random, never derived from public content. Deterministic
framing under that key produces the same ciphertext for identical input.
Addressing by ciphertext ensures that re-ingestion after a content-key shred
uses a different object address. The `blobs/`, `standalone/`, `keys/`, `catalog/`,
`chunks/`, `routes/`, and `obliterated/` namespaces are reserved: depot paths cannot collide
with them and ordinary backfill cannot ingest them as archive files.

## Upload and publication

1. Copy a stable regular-file snapshot while holding an exclusive nonblocking
   lock against supported shim writers. A busy file stays pending; lock
   contention does not exhaust the backend retry budget.
2. Hash the private snapshot and resolve or create the content anchor at its
   current epoch. Register the content as in flight before reclamation can
   decide it is unused.
3. Mint or resolve the path's serving key under the captured upload generation.
   Only publish for that generation; deletion and re-add cannot be overwritten
   by an old upload completion.
4. If the existing path key differs from the shared anchor key, use ordinary
   immutable standalone storage. This is required for mutable paths: a
   create-once per-path key must not be silently replaced when its content
   changes. Correctness takes priority over deduplication savings on rewrites.
5. Encrypt and store the immutable shared ciphertext. Reusing an existing blob
   requires validating its contents, not just observing a successful HEAD.
6. Publish the dedup mapping through the same checksummed causal catalog used
   by packed and ordinary standalone objects, then update local routing and
   mark the matching generation Clean. Local eviction is allowed only after
   durable remote publication.

A dedup catalog entry has `standalone: true` and a `dedup` payload; its object
key is `blobs/<ciphertext_addr>`. All mappings use the mandatory native version 3 catalog envelope. The writer
does not publish a second, independently ordered route mapping.

## Reads and metadata loss

The complete native catalog is the sole shared-blob routing authority. A populated
independent `routes/` namespace is unsupported and refused. Invalid, truncated,
mismatched-prefix, oversized, or unreadable authority fails closed; it is not
absence. The serving key is always the path's current key, so poisoning one path does
not revoke another path sharing the ciphertext.

A read-only cache checks remote authority on every new archive open, including
a local hit. Stale verified Clean bytes remain intact through replacement download and
authentication; only a verified replacement enters the journaled publication step. Unknown
or Dirty local bytes are preserved and the read is refused. Existing open file
descriptors retain snapshot semantics. Writer-server local reads remain fast,
and writer-server eviction remains enabled.

A replacement machine can recover mappings from the complete remote catalog
without the lost local manifest. Preserve the catalog, writer-era tickets,
native role/UUID markers, data, and live key/poison histories in the
recovery plan. A shared blob's address alone cannot reconstruct its referring
paths. New dedup objects do not have independent per-path standalone recovery
descriptors; losing their remote catalog as well as the local state is not a
supported reconstruction path. A catalog rebuild must retain authoritative
existing dedup mappings.

## Obliterate and reclamation

Every deduped object uses its content anchor from its first copy. There is no
lazy promotion and no immediate unique-content exception.

**Per-path revocation:** obliterate durably records its intent, removes local
plaintext, poisons the captured path epoch, and publishes the catalog Remove.
The serving path becomes unavailable once deletion completes. Already-open
file descriptors, application copies, unrelated plaintext caches, and retained
external key backups are outside that revocation guarantee.

**Content-key destruction:** the shared anchor survives until background Tier-2
GC establishes that no path needs it. This applies even if only one path ever
referenced the content. A candidate with no local reference is only a starting
point. Before shredding, GC requires the single-writer attestation, exclusion
of active ingestion, a complete remote catalog and absence of unsupported
independent route records, a complete keystore inventory showing that no other
key slot holds the same key, and authentication of the exact blob under the
anchor. Any surviving reference to either the content hash or ciphertext
address, any other holder of the key, or any incomplete read, defers
destruction. The anchor is poisoned before deletion of its captured ciphertext
object.

GC proves a batch of candidates with one catalog inventory and one keystore
inventory, because neither read depends on which candidate is being proved, and
it performs both without excluding writers. It relies on the single-writer
invariant instead. Every ingest registers its content hash before it can obtain
the shared key, and every catalog publication reports the content hashes and
blob addresses it names; a candidate that either touches while it is being
proved is deferred to a later pass. The exclusive writer drain is held only to
re-confirm the candidate and poison its anchor. A candidate that cannot be
collected is retried with back-off, the object that blocked it is remembered in
memory and re-read alone on retry, and an obliterate ends every back-off.

Unknown orphan blobs are retained. A delayed earlier deletion can otherwise
arrive after the same address has been reused; absence from a local reference
index or an elapsed grace period does not prove this safe. This conservative
policy can grow remote storage. It does not disable normal local cache
eviction or safe reclamation of known content whose key is destroyed first.

## Operational contract

- Exactly one active writer may ingest or obliterate against a backend.
  `dedup_gc_single_writer_ack` records that operator assertion; it is not a
  distributed lock or leader election mechanism.
- `dedup_enabled` controls new dedup uploads. Existing dedup mappings remain
  readable when the flag is disabled. Disabling it parks shared-content GC.
- Use a dedicated keystore with no versioning, soft delete, retention, or
  replication that retains old keys. Retained keys defeat crypto-shred.
- Upgrade daemon and shim together, restart p4d, and fence incompatible writers
  before publishing. A binary-only downgrade after native metadata has been
  published is unsafe.
- Submission acknowledgement is asynchronous with respect to cold storage.
  Losing the writer and its disks before upload completes can lose newly
  acknowledged submissions. Deduplication does not close that durability
  window; zero-loss protection requires synchronous independent durability.

## Confidentiality and deletion limits

While any live path shares content, the shared key and blob remain usable by
someone with raw access to both storage and keys. Deleting one path therefore
revokes that path's access; it does not destroy bytes still needed by another
path. Even the final path's deletion waits for Tier-2 destruction. Operators
requiring immediate cryptographic destruction on every path deletion should
leave dedup disabled.

Content hashes and shared object relationships reveal equality. An observer
with the relevant metadata can test guesses about low-entropy content. The
random content key prevents deriving the decryption key from a guessed hash,
but does not hide equality metadata.

See [the enablement runbook](../enabling-content-dedup.md) and
[disaster recovery](../disaster-recovery.md) for deployment and recovery steps.
