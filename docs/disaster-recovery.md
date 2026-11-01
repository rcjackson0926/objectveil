# Disaster recovery

Upgrading an existing installation? Follow the
[ObjectVeil upgrade and format guide](upgrading.md) before changing
services, state directories, environment variables or cloud prefixes.


ObjectVeil uploads archives asynchronously. A successful Perforce submit is not
an acknowledgement that the new archive is durable remotely. The default
packing deadline is 30 seconds; backlogs, failed writes, and backend outages can
extend this window without a fixed upper bound. Losing the writer and its disks
before upload completes loses the newest bytes unless another independently
durable copy exists. Set the accepted recovery point explicitly; a zero-loss
requirement needs an independent synchronous durability mechanism.

Writer caches reclaim local space by evicting uploaded archives. Unuploaded or
unproven bytes can exceed the configured target and require backlog headroom.
Every authoritative writer to the depot must participate in the shim's write/eviction coordination;
otherwise an external in-place write can race cache reclamation. Deploy the shim
for all supported writer processes, and stop/fence cache mutation during direct
maintenance or archive import by tools that bypass it. Verify this deployment
contract before enabling eviction. Cache limits do not replace backend durability
or a consistent Perforce backup.

Restart p4d when deploying the coordinated-write shim so pre-upgrade writable
file descriptors cannot remain open. Temporary files opened inside the depot
(including p4d's `.tmp.` staging files and `O_TMPFILE`) keep their writer lock
through rename or hard-link publication. Renaming or linking an inode from
outside the protected depot into it is rejected with `EOPNOTSUPP`, including
cross-boundary rename exchange and descriptor-based links. Hard-link exports
are rejected too, since the outside alias could be written without a lock. To import archives,
copy them through normal shim-protected writes in the depot, or stop cache
mutation for a separately controlled maintenance operation. Writable shared
memory mappings and managed `freopen` are also rejected; ordinary guarded opens
and private/read-only mappings remain available. Direct inline system calls,
uninstrumented writers, explicit removal of the writer lock, and mutable
outside hard-link aliases are outside this coordination contract.

Upload snapshots also take an exclusive inode lock while copying source bytes to
private staging, so a writer paused during an in-place edit cannot publish a
stable-looking partial archive. The lock is released before network I/O. Active
writer contention stays retryable and preserves the last durable catalog pointer.

## Recovery dependencies

| Dependency | Purpose and loss consequence |
| --- | --- |
| Primary `chunks/<hash>`, immutable `standalone/<id>` payloads and `.meta` descriptors, deduplicated `blobs/<hash>` | Archive bytes and recovery identity. A missing final copy cannot be reconstructed from an index. |
| Primary `catalog/` deltas, bases, and permanent `catalog/epochs/` writer tickets | Authoritative mappings, deletion records, and causal publication order. Footers and standalone descriptors can reconstruct unambiguous missing mappings. Tickets fence delayed requests from a lost writer and must never be deleted or rolled back. |
| Separate keystore, including live path keys, dedup content-key anchors, and poison markers | Required to decrypt encrypted archives and locate the current epoch after delete/re-add. Loss of a live key makes its ciphertext unrecoverable. Missing poison history can hide later live epochs. |
| Perforce checkpoints/journals and archive metadata | Establish the expected revisions and a consistent server recovery point. objectveil objects alone are not a complete Perforce backup. |
| Configuration, credentials, and backend identity | Reconnect the recovered server to the correct object namespace and keystore. |

The local redb catalog can be rebuilt, but its loss is not proof that remote
objects are garbage. Fresh writers reconcile path epochs from the remote key
history before mutation. Packing is always enabled; files below
`pack_max_file_bytes` normally live in chunks rather than standalone objects.

A configured secondary is a read fallback. objectveil does not synchronously
replicate new primary writes into it. Establish and test any independent
replication process before counting that secondary as a durable backup.

## Key durability and cryptographic deletion

Protect the primary data bucket against accidental deletion using the provider's
retention and versioning features where appropriate. The **keystore is different**:
versioning, soft-delete, WORM/object-lock, historical backups, and key replication
that retain revoked keys defeat cryptographic obliteration. Do not enable these
or restore old key versions as an emergency workaround.

The current design deliberately trades recovery from destroyed keys against
irreversible obliteration. Provision highly durable current-key storage, restrict
administrative deletion, and verify the provider's actual overwrite semantics.
If the required live key is destroyed, surviving ciphertext cannot recover it.
The daemon's startup probes cannot prove that every provider-side retention or
backup policy is absent; deployment policy is part of the contract.

A recovery/replication design for live keys must also propagate revocations and
prove no revoked key remains recoverable. No generic primary-bucket backup
procedure supplies that guarantee. Document and test that policy before relying
on either disaster recovery or permanent deletion.

## Recovering on a replacement host

1. Stop the old writer, or fence its access, and keep the replacement writer
   stopped. Only one writer may mutate a depot. Preserve all surviving primary
   objects and key history; do not run cleanup scripts against them.
2. Install the shipped recovery utilities, restore the configuration and
   credentials, and confirm primary/keystore namespaces and their native role
   markers. Preserve the original namespace UUID. Restore Perforce's database
   to the selected consistent checkpoint/journal position. Initialize the new
   archive-only local root with `objectveil-initialize <config>` (add
   `--existing-archives` only after reviewing an existing archive-only tree).
3. If the catalog was lost, restored incompletely, or its completeness is
   uncertain, run the offline reconstruction while writers are stopped:

   ```bash
   systemctl stop objectveil
   /usr/bin/objectveil-rebuild-catalog /etc/objectveil/config.json
   ```

   The command reads chunk footers, immutable standalone descriptors, and
   authoritative dedup catalog bindings,
   reconstructs local mappings, and publishes missing cloud mappings. Standalone
   recovery streams the stored payload to verify its immutable identity and size;
   it does not buffer an arbitrarily large archive in memory. It does not delete archive chunks. Unreadable objects,
   invalid footers, failed repairs, or unresolved mapping conflicts cause a
   nonzero exit. Partial local work may remain and makes a later retry useful;
   it is not a successful recovery.

   The durable `state_dir/catalog-recovery-incomplete` marker remains until a
   full successful run. Destructive compaction and orphan cleanup refuse to run
   while it is present. Repair the underlying fault and rerun the utility;
   deleting the marker manually does not establish safety.

   Multiple chunks or standalone descriptors can hold different plaintext
   generations for the same path.
   Without a verified authoritative catalog pointer, timestamps and `.seal`
   sidecars cannot establish which revision is current. Reconstruction stops
   with an ambiguity error and preserves all candidates. Recover the expected
   content identity from trusted Perforce metadata or an independently verified
   copy before repairing that mapping. Do not select a winner merely by newest
   timestamp.

   A deduplicated blob does not contain every logical path that references it.
   Preserve its catalog bindings and
   keys; blob bytes alone cannot reconstruct those paths. Current dedup catalog
   entries are authenticated against their referenced blob and key before local
   reconstruction accepts them. A current catalog Remove always overrides a
   historical catalog binding.
4. Verify expected archives end to end against the restored Perforce metadata,
   including member contents, decryption keys, and plaintext digests. Exercise
   cold restores into an isolated read-only cache. A valid chunk checksum or an
   empty upload queue alone is not proof of recoverability.
5. Start the replacement writer only after the recovery command (when needed)
   and archive verification succeed. Record the actual recovery point and any
   missing revisions, verify writer/shim coordination, and size the bounded
   cache and its backlog headroom appropriately.

With a complete healthy catalog, read-only caches can bootstrap mappings lazily
without a full reconstruction. A local incomplete-recovery marker blocks daemon
startup, including read-only startup, until reconstruction finishes. Ordinary cold
lookup does not search every chunk footer automatically; offline reconstruction
supplies that missing index.

Read-only refresh keeps an old verified local generation while downloading and
authenticating its replacement. Durable records in `state_dir/refresh-retained/`
prevent ordinary eviction from discarding that copy after failure, cancellation,
or restart. A later successful authenticated refresh or verified current warm
read clears the record. Retained files can keep the cache above its target;
removing these records manually does not establish remote recoverability.

The short final replacement step uses the existing eviction journal and held
inode. A crash before replacement can restore the old inode automatically.
If recovery finds both a held old inode and a different public replacement,
it preserves both and refuses startup for manual reconciliation. Preserve the
eviction journal and both files, verify them against authoritative remote and
Perforce metadata, and resolve the conflict before restarting. Do not delete
one merely because its timestamp is older.

Successful reconstruction prints counts for scanned chunks, scanned standalone
objects, recovered mappings,
re-emitted mappings, incomplete shards (zero on success), adopted pointers, and
chunks that initially lacked cloud references. Success establishes completion
of the observed reconstruction work, not that every revision expected by
Perforce exists; the independent verification step is still required.

Interrupted deletion keeps a durable intent pinned to its original key epoch.
New publication for that path waits until the old deletion and cloud tombstone
complete. The local plaintext unlink and its parent directory must be durable
before success is acknowledged; a failed directory sync keeps the intent even
if the next retry sees the file already absent. Retry and startup recovery
preserve a newer local generation and any already-published replacement key. Persistent catalog failures therefore delay
re-add publication rather than allowing an old deletion to overtake it.

Opt-in backfill can consider explicitly prepared unframed path-addressed
candidates only inside an already identified native namespace. Cloud candidates
require an independently recorded plaintext hash. Without it, bytes might be
damaged ciphertext, even if the keystore is missing; backfill retains them and
reports a blocked pass. This also applies when an earlier publication minted a
key but failed before publishing its chunk. Verify source identity against
authoritative archive metadata before repair. Backfill excludes native framed
and `standalone/` payloads and does not convert a foreign namespace or catalog.

## Cleanup and upgrade behavior

Physical chunk and unknown-blob sweeps are disabled. Even a complete catalog
and validated replacement payloads cannot prevent a delayed DELETE from removing
a later publication at a reusable address. Superseded and unmapped chunks, seal
markers and recovery metadata remain stored until a durable address-retirement
protocol exists. Missing mappings, unreadable metadata and damaged footers never
establish that retained bytes are disposable. Account for remote storage growth.
Recovery's incomplete marker also prevents ordinary spatial repacking from
acting on partially reconstructed liveness counts.

Each writer reserves a permanent, atomically created ticket in `catalog/epochs/`
before publishing. Deltas and bases carry that writer era; within an era, each
shard uses an increasing order seeded once from complete remote history. Folds
retain a constant-size causal watermark rather than every historical delta name.
Legacy delta names are retained only while a verified complete listing still
contains them. This prevents failed old-object deletion or a delayed request from
a lost host from overriding a newer publication, including after another fold.

Large new archives use immutable `standalone/<id>` payloads with immutable `.meta`
recovery descriptors. The payload identity binds the logical archive path and
stored-byte digest. The descriptor is durable before its catalog pointer, and the
pointer is durable before the local row becomes Clean. Old full-object versions
and their descriptors are retained conservatively; object storage usage can grow
after rewrites or deletion. Current cleanup does not reclaim these versions
without a separate proof of safety. Crypto-shred still revokes their shared path
key epoch; plaintext NFS storage does not provide permanent physical obliteration.
Tombstones remain while a descriptor could reconstruct the deleted mapping.

Deduplicated archives publish through the same causal catalog. Mutable rewrites
retain existing path-key custody and can fall back to standalone storage when
sharing another anchor would invalidate earlier content. Shared-blob deletion
requires complete remote reference evidence, authentication under the exact
content anchor, and a check for other live custody of that key. Unknown orphan
blobs are retained: a delayed deletion of a deterministic blob address could
otherwise destroy a later reuse. This deliberately favors recoverability over
immediate space reclamation or unconditional cryptographic-erasure completion.

Catalog records use checksummed native version 3 with format `objectveil-catalog`.
Other versions are refused. Native storage identity markers bind both roles to
the configured namespace UUID. Preserve those markers alongside objects.
Upgrade every reader and writer together and fence old binaries before publishing
with the new ordering and standalone pointers. Preserve all writer tickets and
recovery descriptors in primary-storage backups. A catalog restored to an older
point must not reuse or discard newer tickets. The protocol requires a single
active writer, atomic create-once support, and strongly consistent complete object
listings. This does not provide live multi-writer coordination. Lost catalog
history containing several plaintext versions remains ambiguous and requires
trusted external archive identity even though all candidate bytes survive.

Empty framed encrypted objects now contain an authentication tag. Earlier
header-only empty objects have no proof of key possession and are rejected.
Recover those objects from independently trusted empty archive contents and
upload them using the corrected writer; do not bypass authentication to migrate.

## Read-only cache freshness

Read-only shim processes query the daemon's role once and retain that role for
their lifetime. Restart the shim-bearing `p4d` process whenever its cache role
or deployment configuration changes. Every new read-only open starts a fresh
authority check against the complete remote catalog and the current key epoch.
It verifies the local inode and full plaintext digest before allowing a warm
hit; a newer remote revision is downloaded and authenticated while the verified
tracked Clean local copy remains protected from eviction. Only successful
replacement publication retires that copy, using the crash-recoverable eviction
journal for the final swap.
Untracked, Dirty, actively written, or changed local bytes are preserved and the
read fails so they can be investigated.

A backend/catalog/keystore outage fails these reads instead of serving unchecked
warm bytes. This adds metadata requests and full-file hashing to read-only warm
opens, and each read uses the progress-poll protocol. Each daemon admits at most
16 concurrent validations, including unfinished blocking hashes; overload returns
a retryable refusal instead of queuing unbounded work. An already-open file
descriptor remains a snapshot; the cache cannot forcibly revoke bytes a client
already opened or copied. These guarantees require all read-only archive access
to use the updated shim and all cooperating processes to follow its locking
protocol.

Native version 3 catalog deltas and bases include a canonical JSON checksum
covering causal order, deletion records and payload pointers. Valid-JSON bit
corruption therefore fails closed instead of silently selecting an older
archive. Other catalog versions are refused; online migration is not an
incompatible-format converter. The checksum detects accidental corruption; it
is not a signature against an attacker able to replace the object and recompute
its digest.

Full cache verification checks stored archive identity and authenticated contents
against the catalog. It does not independently prove every logical Perforce
revision inside an RCS archive. Recovery acceptance must also run actual
`p4 verify` against restored Perforce metadata and the recovered archive set.

## Related

- [Packing configuration](enabling-chunk-packing.md)
- [Operations](OPERATIONS.md)
