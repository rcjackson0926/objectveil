# Object-store requirements and bounded transfers

ObjectVeil uses cloud object storage as durable archive backing, with separate
catalog authority and key custody. This document describes the native runtime
contract; [configuration](../CONFIGURATION.md) lists accepted provider options.
Release qualification requires the exact build's evidence and is not established
by this design description.

## Namespace ownership

Configuration requires `format: "objectveil/1"` and a canonical non-nil
`namespace_id`. Primary and keystore each contain a role-specific marker at
`catalog/objectveil-format-v1.json`, with format `objectveil-native-storage`,
version 1 and the same namespace UUID. Native wrappers validate the paired
identities during operations. A writer can initialize only empty namespaces;
a missing partner marker can be completed only while both sides are empty or
marker-only. Read-only clients never initialize stores.

Missing, malformed or mismatched identity is an error, not an empty cache.
Populated incompatible storage requires the separately controlled conversion
workflow in [upgrading](../upgrading.md). Cloud prefixes and NFS object keys are
opaque storage identities; they must not be decoded into user filesystem paths.

## Data and key durability

Cloud primary storage requires a separate keystore. The configuration refuses
ambiguous same-bucket arrangements. Primary data retention may preserve immutable
payloads and catalog history. Keystore policies must permit effective revocation:
versioning, soft-delete, immutable retention, historical key backups and replication
that retains revoked keys can defeat crypto-shred.

Startup probes test the create-once and overwrite behavior visible through the
storage interface and examine exposed version identifiers. They do not establish
that provider backups, retention, replication or administrator copies are absent.
The operator must verify those policies for every provider. GCS attestation is
needed because the daemon's current interface does not expose enough policy
information; it does not imply that GCS has no policy API.

Loss of a live key makes encrypted data unrecoverable even when every payload
survives. Recovery must preserve live keys and poison histories while preventing
revoked key recovery. See [crypto-shred](obliterate-crypto-shred.md) and
[disaster recovery](../disaster-recovery.md).

## Publication and reads

Uploads write immutable payloads and durable native catalog authority before the
captured local generation becomes Clean. Payload existence alone is insufficient.
Catalog reads require complete checked inventories; uncertain or truncated reads
must not become absence or permission to delete. The current catalog contains
standalone, packed and deduplicated mappings. An independent populated `routes/`
namespace is unsupported.

Read-only cache hits still check current authority. Cold reads authenticate bytes,
size and keys before publication. Packed reads can coalesce several archive
requests into one chunk download, and range requests are pinned to the selected
object identity. Stored-object limits include authentication and footer overhead;
logical plaintext limits remain independently enforced.

A secondary is a fallback source, not synchronous replication or an immutable
backup. Ordinary uploads target primary. Maintenance can remove secondary data,
so its permissions and independent retention must be chosen deliberately. Both
migration writer and verification clones clear secondary to prevent hidden
fallbacks or mutations outside the migration destination.

## Multipart and local resource limits

The shared cloud uploader uses four concurrent 8 MiB part buffers, with at most
10,000 parts per object. It verifies the actual source length, refuses an
oversized stored object before upload and handles short/growing input as failure.
Cancellation and failed completion attempt to abort the multipart upload.
SIGKILL, power loss and provider failure can still leave parts; provider lifecycle
cleanup of incomplete uploads is an operational requirement.

Small objects use a single PUT where the provider multipart minimum requires it.
Packing is always available for eligible files; thresholds choose packed versus
standalone publication. Incompressible members can use STORE codecs rather than
paying for ineffective gzip. Footer size is included in admission and sealing,
including the bounded single-entry overshoot.

Migration separately accounts retained workspace bytes, payload reservations and
actual database growth against its workspace budget. Its runtime drains blocking
operations before releasing the job lock. These bounds do not reserve free disk
against unrelated processes; allocation failures still fail the operation safely.

## Storage classes and cost

`primary.storage_class` applies the validated write-time tier to new data PUTs,
including multipart uploads. It does not rewrite existing objects. Per-request
class overrides, keystore classes and secondary classes are refused. Only
supported immediately readable classes are accepted; provider retrieval charges,
minimum retention and minimum billed object sizes still apply.

Lifecycle rules can manage primary data tiers and incomplete multipart uploads,
but must preserve immediately readable archives and the catalog/key dependencies.
Do not apply data lifecycle deletion rules to the keystore. Match compute and
storage region deliberately; a region advisory is not an egress-cost guarantee.

Backend request and byte counters provide operational visibility. Identity checks,
catalog reads, key checks and freshness validation add requests even for warm
reads. Cost planning should use measured workloads and current provider pricing.

## Reclamation limits

Compaction authenticates bounded input before publishing a replacement mapping;
old content is not removed on an incomplete read. Known shared-content reclamation
requires complete authority and key-custody proof under a fenced sole writer.
Unknown orphan blobs are retained because delayed deletes and address reuse can
make apparently unreferenced content unsafe to remove. Retained ciphertext,
metadata and provider history can therefore consume storage after logical deletion.
