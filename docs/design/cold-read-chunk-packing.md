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

## Compaction, backfill and deletion

Compaction identifies members still referenced by complete current authority,
downloads the old chunk within the actual stream cap, validates bounded local
reads and hashes, and publishes replacement chunks and catalog pointers.
Verbatim ciphertext can be repacked without exposing neighboring plaintext.
Original chunk payloads, seal markers and recovery metadata are retained,
including fully superseded chunks. A timed-out DELETE can execute after identical
content is published at the same address; current liveness and process locks do
not prevent that race. Physical chunk and unknown-blob sweeps therefore perform
no deletion or inventory requests until a durable address-retirement protocol
exists. Storage consumption can grow; repacking does not reclaim remote bytes.
On size refusal or failed publication, original objects remain intact.

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
