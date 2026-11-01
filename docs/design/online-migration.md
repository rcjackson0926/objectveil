# Online archive migration architecture

Status: sealed-source and direct live-source import are implemented. The uploader
now uses ordinary cache packing as well as multipart transfer; reverse copying
is implemented with `--download`. See the
[migration runbook](../migration.md) for the supported workflow and the exact
release's published validation receipt for its tested acceptance scope.
Release acceptance requires the live-source adversarial tests, required Linux
gate and packaged-runtime checks to pass for the exact artifacts. This design
document does not claim those checks passed for a pending candidate.

## Required outcome

Copy an existing Perforce installation's physical archives into the exact
encrypted object, key and catalog format consumed by ObjectVeil. The primary
Perforce service must remain online. Preserve the original archives. Resume
after interruption and verify every copied archive through a fresh cache.

Perforce databases, checkpoints, journals, configuration and credentials remain
separate backup responsibilities. Copying arbitrary P4ROOT contents into object
storage does not establish a recoverable Perforce installation.

## Online source consistency

The default workflow copies directly from an actively used physical archive tree.
It does not require a replica, filesystem snapshot, full tree copy or modified
Perforce processes. It may skip busy files; continuously following changes is
outside the requested finite import.

Advisory locks and stable timestamps cannot distinguish a completed archive from
a paused writer holding an open descriptor. Live mode therefore requires Linux
kernel read leases on an explicitly accepted local filesystem: ext2/3/4, XFS,
Btrfs or tmpfs. Refuse network, FUSE, overlay and unknown types even if a local
lease call succeeds, because that does not prove distributed writer exclusion.
Archive ownership or `CAP_LEASE` and normal source read/traversal permissions
are prerequisites.
Unsupported leases are a hard error, never permission to use a weaker fallback.

Inventory hashes one file at a time under a lease without retaining its bytes.
A worker subsequently reacquires a lease and copies only the expected file
generation into bounded private staging. Existing writers or writable shared
mappings cause a skip. A writer requesting access while capture runs causes the
capture to be abandoned promptly. Descriptor identity, content hash and a fresh
resolution of the source pathname and ancestors guard replacement and rename
races. Leases cover local source capture only, never encryption, network upload
or cold verification.

Live traversal tolerates unrelated directory membership changes and skips busy,
changed or disappeared files without suppressing successfully captured neighbors.
It preserves no-follow confinement, special-file rejection, byte-exact names,
bounded directory buffers and depth limits (100,000 entries per directory,
16 MiB of buffered names, depth 128 and 4 KiB relative paths). Bound failures are
fatal, not source-busy deferrals. Root ownership binds its path, device
and inode rather than mutable directory timestamps.

The durable job can be rerun against the same live root and destination. Each
pass revisits current files, discovers new entries and invalidates previous
verification when a file generation changes. Progress and skips describe the
current pass, so old or vanished ledger records cannot inflate completeness.
Immutable object publication and causal catalog ordering handle replacement
archive contents; the tool does not issue remote deletes for missing source
paths. All other cloud writers remain excluded during migration.

A live result verifies individually captured file generations, not a single
transactionally consistent Perforce server generation. Files may change or be
removed after capture. The result grants no permission to remove local archives
or omit metadata backups. Whole-server cutover requires separate reconciliation
and verification.

Explicit `snapshot` and `replica` modes retain the stricter sealed-generation
contract. Their complete source must remain unchanged through final inventory
validation. A live pass does not claim that contract, and operators need not
prepare those alternative sources for ordinary live import.

## Native destination and durable execution

The destination configuration must declare `objectveil/1` and an explicit
namespace UUID. Paired primary/key role markers, native catalog authority and
dedicated job ownership bind the destination. Existing incompatible data cannot
be silently adopted; a private external conversion process must establish the
native format before a runtime uses converted data.

The migration journal records source/configuration identity, per-file expected
generations and verified progress under an exclusive process lock. The private
writer reuses ordinary key custody, packing, immutable publication and shared
multipart transfers. It starts no eviction, backfill or garbage collection;
both writer and cold-verification clients have secondary fallback disabled.

The workspace budget accounts retained files, actual database growth and active
payload reservations. Successful worker reservations remain charged until the
whole wave drains and retained storage is recounted. Cancellation retains the
session lock and reservations until the runtime's blocking work finishes.

Publication is verified through a separate read-only engine using the expected
archive size and digest. A queued upload or a successful PUT is not a verified
receipt. License features, effective expiry, capacity and watermark policy apply
before publication; credentials are not stored in the resume identity.

The packaged migration and initializer executables perform runtime integrity
checks. The initializer establishes an archive-only root proof, not a snapshot
or data conversion. See the runbook for import, reverse copying, restart and
cutover commands.

## Adversarial acceptance coverage

The direct-live extension requires the following acceptance matrix. In particular,
exercise actual uninstrumented writers and Perforce processes; advisory-lock-only
fixtures cannot establish live-source safety.


- Same-size rewrites, paused writers, rename/replacement, removed/new files,
  directory substitution, symlink escapes, FIFOs/devices, inaccessible entries,
  non-UTF-8 names and reserved logical keys.
- Missing archives and historical RCS revisions, shelves, custom depot maps,
  unsupported special depots and incomplete metadata. Unsupported scope must
  block claims of whole-server completeness.
- Concurrent jobs, foreign/aliased state, changed roots or target prefixes,
  nonempty destinations, partial listings, lost local state, and stale resume
  records after a remote publication or local commit failure.
- Interrupted encryption, key creation, multipart transfer, payload/descriptor/
  catalog publication and final verification. Preserve source data and reject
  missing, corrupt, truncated or unauthenticated remote content.
- Multipart overlap and bounds, final-part/complete errors, cancellation,
  disk exhaustion, license expiry/renewal and capacity refusal.
- Cold restoration with no upstream Perforce fallback. A healthy upstream must
  not hide a missing cloud archive during verification.

Existing asynchronous writer durability limits, single active writer fencing and
provider key-retention requirements continue to apply after migration. Acceptance of a release requires independent review and actual validation
evidence for its exact source and packaged binaries.
