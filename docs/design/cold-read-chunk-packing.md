# Cold-read chunk packing

Packing combines small physical archive files into immutable primary objects.
Large files use standalone storage. Packing reduces requests; it does not change
which logical path, content hash or key epoch is authoritative.

## Representation

A chunk contains concatenated member bytes, a JSON table-of-contents footer and
a trailing four-byte little-endian footer length. The footer's supported version
is `1`. Each member records storage-key bytes, offset, stored length, plaintext
length, plaintext BLAKE3 and a codec. The codecs are gzip, authenticated gzip,
verbatim storage and authenticated verbatim storage. Already-compressed archive
names can use the verbatim forms. Empty members are valid.

A fresh plaintext cloud namespace writes an eight-byte `OVPACKU1` signature
and a random 16-byte publication UUID ahead of the first member. Footer offsets
include these 24 bytes. Each seal has a distinct immutable address, even when
all member paths and bytes are identical to an earlier seal.

The enclosing native storage namespace and version 3 catalog identify this
format generation. No envelope from another catalog generation is accepted.
Internal names and escaped path keys remain distinct; NFS stores object keys
opaquely instead of interpreting a key as a decoded depot pathname.

A member's footer hash describes the original archive bytes. An encrypted codec
needs the path key authorized by the current key-epoch history. A chunk can contain unrelated
members and keys; deleting one member does not require decrypting or rewriting
its neighbors.

## Publication and limits

The engine captures a stable private source generation before hashing or packing.
Members are grouped into packing buckets. Seal triggers include target size,
flush deadline and explicit drain. Admission accounts for the entire encoded
footer as well as stored member bytes; many tiny or empty members cannot grow
unbounded metadata outside the configured chunk target. A bounded single-entry
overshoot covers the largest escaped key and member. The footer length must fit
its on-disk integer representation.

L0 admission holds at most 64 open or sealing buckets and 256 MiB of conservative
member, buffer and footer reservations. A bucket seals at 64 MiB of reserved
memory even when the configured target is larger. In-memory capture is capped
at 16 MiB; files at or above the effective threshold use streaming native
standalone publication. Saturation leaves work durably Dirty with persisted
backoff. These are packer resource bounds, not a total-process RSS guarantee or
a substitute for the separate multipart and restore bounds.

The chunk is published before its catalog Put; only the matching local generation
becomes Clean after durable remote authority. A failed catalog publication retains
local recoverable data. A crash may leave an extra immutable object. Local upload
success alone is not permission to evict before publication completes.

Readers validate whole-object and member extents before allocation, limit streamed
bytes even when HEAD metadata lies, bound decompression and verify plaintext
length and BLAKE3. The packed decoder caps one uncompressed member at 1 GiB;
configuration rejects larger packed-file thresholds. Oversized, truncated,
unknown-codec and corrupt members fail closed without publishing a local file.

## Read path

The current catalog selects the immutable chunk and exact member. Small reads
can use bounded ranges; coalesced reads can fetch a complete chunk once and
prefetch authenticated neighbors subject to resource limits. The requested member
must match authoritative size and hash. A corrupt encrypted member cannot silently
become raw plaintext, and missing catalog authority cannot be repaired by guessing
from an obsolete object at the logical path.

The private staging file is synchronized before local publication. A read-only
cache refresh retains its previous verified inode through remote GET and
authentication failure. The final replacement window is covered by the local
eviction/held-inode journal. Existing open descriptors keep their old snapshot.

Loose files directly under the archive root share one physical packer. So do
the revision files in top-level Perforce `name,d` directories. Other trees
retain their top-level directory grouping. The existing catalog lookup for
each archive stays the same when its physical pack group changes.

## Compaction, backfill and deletion

Compaction identifies members still referenced by complete current authority,
downloads the old chunk within the actual stream cap, validates bounded local
reads and hashes, and publishes replacement chunks and catalog pointers.
Verbatim ciphertext can be repacked without exposing neighboring plaintext.
In plaintext cloud mode, obliterate records a durable catalog tombstone and
removes the member's local catalog reference. The periodic compactor processes
chunks below `compaction_live_fraction` and fully dead chunks. For each eligible
uniquely published pack, the writer checks that every original member has a
durable catalog tombstone or a readable and verified replacement. It records
the retirement locally, removes the primary seal and payload, and confirms their
absence. A failed or interrupted delete is retried against the same single-use
address. Replicas, external backups, provider snapshots and soft-deleted versions
have separate retention. Restoring a historical catalog requires matching
historical copies of its referenced objects.

A pending plaintext delete records its originating catalog writer era before
touching the archive. An ordinary restart checks the immutable tickets and
publishes the tombstone under that same era. If a later writer already took
over, reconciliation leaves the old local file and intent intact and reports
the conflict. Keep the superseded state offline. Check the path on the active
Perforce writer, issue any still-required deletion there, and recover a cache
from the current namespace with fresh local state.

Reusable legacy and encrypted pack addresses are retained. A delayed DELETE
could otherwise remove an identical later publication. Untracked objects and
missing or incomplete catalog evidence never authorize reclamation.

Backfill imports eligible raw primary objects into packing only within the
configured native namespace. Both the actual download and local read are bounded
by the per-file packing limit, including growth after HEAD. It excludes internal control prefixes, verifies
ownership and source identity, and respects poisoned path epochs. A secondary
may be a migration source and cleanup target; it is not an immutable backup.

Encrypted obliterate revokes the path key and publishes a tombstone. Packed
ciphertext can remain physically present while the key is inaccessible. This
requires a keystore that cannot recover overwritten keys. An unencrypted NFS
archive does not gain cryptographic erasure from packing.

## Recovery boundary

Chunk footers support reconstruction, but cannot decide between conflicting
historical generations by themselves. Surviving catalog tombstones and causal
records take precedence. Rebuild refuses incomplete scans or ambiguous authority.
Keep catalog, payload and key dependencies together with independent Perforce
metadata backups; a collection of chunks is not a complete server backup.

See [catalog](catalog-shard-crdt.md), [crypto-shred](obliterate-crypto-shred.md),
[configuration](../CONFIGURATION.md) and [packing operations](../enabling-chunk-packing.md).
