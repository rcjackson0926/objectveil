# Obliterate and crypto-shred

## Current native contract

An encrypted archive uses a random per-path, per-epoch key in a separate keystore.
Obliteration overwrites that key with an exact poison marker. Ciphertext encrypted
under the revoked key becomes undecodable provided no recoverable copy of the key
survives. The primary may retain old immutable standalone objects and packed
chunks; those encrypted bytes do not need to be found and overwritten individually.

The local depot cache contains **plaintext**, not ciphertext. The deleting daemon
must durably remove its local plaintext before acknowledging deletion. Read-only
peers check remote catalog and key authority before each new shim-mediated open;
this refuses retained plaintext after revocation. It does not erase every peer's
local disk, revoke already-open descriptors, or destroy plaintext previously
copied outside the cache. Unencrypted NFS deployments do not provide cryptographic
obliteration. See [the recovery runbook](../disaster-recovery.md) for these limits
and the required writer/shim deployment contract.

## Encryption and key identity

- Encryption happens before cloud publication. Packed members use authenticated
  codecs; immutable standalone payloads use framed streaming authentication.
  `content_hash` describes plaintext and is checked after decryption.
- Epoch zero uses the path-addressed key slot. Later epochs use a separate,
  collision-free namespace. A serve resolves the contiguous poison history to
  the first live key; replacement writers reconcile that remote epoch before
  mutation. Only the exact poison marker authorizes advancing to another epoch;
  malformed slots and transient reads fail closed.
- Keys are created atomically once. A genuine re-add uses a new epoch and key;
  captured older upload generations may not retag a newer local row. Pending
  deletion prevents a new publication from overtaking its durable Remove record.
- Authenticated empty framed objects contain an empty frame tag. Header-only or foreign-format objects are refused. Damaged framing cannot
  silently become plaintext; conversion requires independently trusted identity.
- A secondary is a read fallback, not synchronous replication. Ordinary new
  uploads target primary; maintenance may remove migrated or obsolete secondary
  objects. Do not treat the secondary as an immutable independent backup without
  a separately established policy.

## Durable deletion protocol

The daemon takes the obliterate write drain so cooperating local uploads,
backfill, restoration and eviction cannot cross its deletion boundary. The path's
selected old epoch remains the retry identity throughout the operation.

1. Refuse unresolved eviction journals for this path. Reconcile remote epoch
   authority, persist the pending deletion intent and advance local path epoch
   before revoking the old key.
2. Remove the old local plaintext and synchronize its parent directory. A real
   error keeps the intent and refuses acknowledgment. An already-absent retry
   still synchronizes the surviving parent. A provably newer local generation is
   retained instead of being deleted by the older intent.
3. Overwrite the selected old epoch's key with poison. Backend failure leaves the
   operation retryable. Poison remains present, so a delayed create-once request
   cannot mint a replacement key into an empty old slot.
4. Remove the obsolete path-addressed payload, publish the durable catalog Remove,
   and retire the corresponding local catalog entry. These are required steps,
   not best-effort post-acknowledgment cleanup. If remote state proves a newer
   already-published re-add, preserve that newer key and catalog instead.
5. Remove only the obsolete local FILES generation and clear the pending intent
   last. A retry or startup reconciliation resumes the same old epoch.

The intent, epoch gate, directory synchronization and tombstone are all part of
the current contract. Failures return an error; the daemon retries
persisted intents on recovery, and the caller must retry an unacknowledged delete.

## Keystore durability and irreversible deletion

A successful overwrite must actually revoke all recoverable copies of that key.
Keystore versioning, soft-delete, object-lock/WORM, retained historical backups,
and replication that preserves revoked keys defeat this property. The primary
data bucket can use separate retention/versioning policies.

Startup probes check create-once, overwrite/delete behavior visible through the
backend, and exposed version identifiers. S3/Azure version identifiers can reveal
enabled versioning. GCS requires explicit operator attestation because the current daemon storage
interface does not expose enough retention policy information. Provider APIs
exist for operator verification; object generations alone do not establish
whether historical versions remain available.
These probes cannot prove hidden provider backups or retention policies absent;
that remains a deployment requirement.

Conversely, losing a live key makes surviving ciphertext unrecoverable. Recovery
must preserve current live keys and the poison epoch chain without restoring
revoked key material. A recovery/replication policy must propagate revocations and
establish that old keys cannot be recovered. See the recovery runbook before
claiming both disaster recovery and permanent deletion.

## Rebuild and cleanup

Offline rebuild scans chunk footers and immutable standalone descriptors. A
complete surviving catalog Remove suppresses reconstruction of that path.
Different historical plaintext generations require authoritative surviving
metadata or trusted external identity; recovery refuses to guess by timestamps.
Incomplete reconstruction leaves a durable startup/cleanup prohibition.

The latest catalog Remove remains until a newer Put supersedes it. There is no
time-based tombstone pruning; any future pruning would require a separate proof
that retained recovery metadata cannot resurrect the path. Ordinary orphan cleanup requires a complete durable
Remove or a validated replacement mapping/payload; missing catalog metadata
never establishes that recoverable bytes are garbage.

Old immutable standalone versions and descriptors currently remain in storage.
Their retained encrypted bytes are protected by revocation; retained plaintext
NFS versions remain recoverable and are not a compliance deletion guarantee.
