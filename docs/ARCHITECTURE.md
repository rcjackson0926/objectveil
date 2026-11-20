# ObjectVeil architecture

ObjectVeil maintains a local plaintext archive cache backed by immutable objects,
a durable catalog and, for encrypted deployments, a separate keystore. Perforce
is the qualified application integration. Other applications need their own
write, deletion and completeness qualification; see
[application qualification](design/perforce-coupling.md).

```mermaid
flowchart TD
    P[Perforce archive operations] --> S[libobjectveil_shim.so]
    S -->|OBJECTVEIL/1| D[ObjectVeil daemon]
    W[Coordinated writes and fanotify] --> D
    D --> M[Private manifest and recovery journals]
    D --> H[Local plaintext archives]
    D --> B[Primary: catalog and immutable objects]
    D -. encrypted deployments .-> K[Separate key namespace]
    S --> A[Versioned access datagrams]
    A --> L[Bounded access logger and optional spool]
```

## Native identity and startup

Configuration requires `format: "objectveil/1"` and an explicit canonical,
non-nil `namespace_id` UUID. That identity is immutable for a running process.
The default state path is `depot_path/.objectveil`; explicit state paths are
subject to the same ownership validation. Populated unmarked state is refused.

`objectveil-initialize` checks an empty archive root, or an existing archive-only
root explicitly attested with `--existing-archives`, before publishing local
ownership and archive-root proof. It does not copy archives, establish a source
snapshot or convert incompatible state. The daemon requires that proof before
license anchors, manifests or watchers can write. See [upgrading](upgrading.md).

A new plaintext cloud writer claims an empty primary with a single
`catalog/objectveil-format-v1.json` marker. The marker binds the namespace
UUID and storage mode. Its version is 4 for plaintext partitioned storage or 3
for plaintext legacy storage. Encrypted deployments also validate the original
keystore marker. A partially initialized pair can resume only while both
locations are empty except for their markers. Read-only clients cannot initialize stores.
Missing, malformed, foreign-role or foreign-UUID identity fails closed.

The shim requires canonical `OBJECTVEIL_*` configuration and `OBJECTVEIL/1`
requests. An absent managed root is a startup failure unless the shim is
explicitly disabled. There is no unversioned request or alternate-format decoder.
Socket pointer files and access datagrams also have mandatory native headers.

## Durable state and authority

| State | Purpose |
| --- | --- |
| Local `manifest.redb` | File generations, durable dirty work, LRU, pending deletion/eviction recovery and catalog mirrors. Native file records use the `OV` binary discriminator. |
| Remote catalog | Checksummed, ordered Put/Remove records for packed, standalone and deduplicated objects. Legacy v3 groups by leading directory. Partitioned v4 uses the full archive key, immutable base segments and a version-fenced storage marker. The local catalog mirrors remote records. |
| Writer-era tickets | Permanent create-once records ordering a replacement writer after prior writers; not leader election. |
| Primary payloads | Immutable standalone objects, packed chunks and shared encrypted blobs. |
| Separate keystore (ObjectVeil encryption only) | Live path/content keys and exact poison histories; loss of a live key loses encrypted data. |
| Migration/verifier state | Dedicated owned directories and locks, separate from the daemon manifest. |
| Access spool | Optional owned, bounded telemetry persistence; never an archive durability substitute. |

A complete catalog Remove suppresses restoration and reconstruction. An incomplete
or corrupt authority read is not absence. Dedup routing lives in the catalog;
a populated independent `routes/` namespace is unsupported and refused.

## Write, restore and eviction

Supported shim writers coordinate open descriptors with eviction. The watcher
records dirty observations durably and periodically reconciles the archive tree.
An upload captures a private stable generation, resolves its encryption key when configured, publishes
immutable data and durable catalog authority, then marks only that generation
Clean. Uploads target the primary; a secondary is not synchronous replication.

Writer reads can use valid local data. Read-only caches check current remote
authority on every new managed open, including local hits. An unknown or Dirty
local file is preserved and refused. A stale verified local file remains intact
while its replacement is downloaded and authenticated. Only successful private
staging permits replacement; the publication window is journaled for recovery.
Already-open descriptors retain their existing inode contents.

Cold FETCH work is detached and bounded; cancellation of a waiting RPC does not
release a still-running restore's admission. FETCH and writer STAT share the
configured worker bound. Versioned asynchronous STAT uses bounded tickets and
expiry. The shim applies one total operation deadline across connection, polling
and retries. Readiness requires the full engine; socket existence alone is not
readiness.

Eviction requires a Clean matching generation, verified recoverable remote
content and exclusion of active writers. A durable held-inode journal covers the
local unlink and metadata transition. Startup reconciles incomplete operations
before admitting unsafe work. Read-only mode still evicts its local cache; it
rejects managed writes and authoritative deletion.

## Deletion and recovery limits

Deletion persists a generation-specific intent, removes the local archive, and
publishes a durable tombstone for that archive path. In plaintext cloud mode,
other paths in the same pack remain readable. The compactor can retire the
old pack after publishing and verifying every surviving member. A fresh
publication uses a new pack address even if the contents are identical.

In encrypted deployments the deletion also poisons the path key epoch.
Optional content deduplication can give two paths a shared blob. The shared
content key can be destroyed only after the last reference is proved gone.
See [crypto-shred](design/obliterate-crypto-shred.md) and
[content deduplication](design/content-dedup.md).

Perforce submit acknowledgement is asynchronous with respect to cloud publication.
Total loss of the writer and its disks can lose acknowledged but unpublished
archives. Recovery needs the surviving catalog, payloads and writer-era tickets, plus live keys and poison
histories for encrypted deployments, and independent Perforce metadata backups.
Checksums are corruption checks, not authenticity against a storage administrator
who can replace both data and checksum. Provider retention or key backups can
defeat permanent erasure.

## Distributed tools

The package contains eight executables and the matching preload library:
`objectveil`, `objectveil-initialize`, `objectveil-migrate`, `objectveil-verify`,
`objectveil-manifest-summary`, `objectveil-rebuild-catalog`, `objectveil-rescan`,
`objectveil-dedup-measure`, and `libobjectveil_shim.so`. The daemon, initializer and
migration executable include runtime integrity checks. Release acceptance applies
to the exact tested binaries and evidence, not this architectural description.

See [configuration](CONFIGURATION.md), [operations](OPERATIONS.md),
[migration](migration.md), [tuning](TUNING.md) and
[disaster recovery](disaster-recovery.md).
