# Perforce integration and application qualification

ObjectVeil is a file/object-tiering platform. Its storage engine works with
relative file identities, immutable payloads, a current catalog, separate
access keys and local residency. That abstraction is useful beyond a version
control server, but it is not a promise that every application can safely use
the current preload shim or its deletion semantics.

Perforce is the integration exercised by the current end-to-end tests and
operator tools. Qualify each additional application before exposing production
files or allowing eviction. A successful file copy or an intercepted read is
not sufficient evidence of correct concurrent writes, deletion or recovery.

## Integration boundary

| Layer | Shared mechanism | Integration work that remains |
| --- | --- | --- |
| Storage and catalog | S3/Azure/GCS/NFS backends, immutable payloads, packing, content hashes and current authority | Provision separate key storage; define durable publication and recovery dependencies |
| Local residency | A bounded local cache and verified restoration | Establish which files are managed and how the application discovers missing files |
| Preload shim | Linux libc interposition, path scoping and daemon requests | Validate the application's actual calls, descriptor lifetimes, mappings, permissions, forks and error handling |
| Writes and eviction | Coordinated writer opens/closes and a fenced writer | Ensure every writer participates; validate in-place writes, temporary files, atomic renames and deletion races |
| Perforce archive handling | Physical relative archive paths | Perforce archive layouts, shelves, RCS updates, archive-root Maps and `p4d` behavior remain integration-specific |
| Verification and rescan | Independent completeness checks | Current checkpoint/journal parsing and `p4 verify`/storage checks are Perforce-specific; another application needs its own authority |

The environment variable `OBJECTVEIL_DEPOT` selects the managed physical root.
It is not evidence that arbitrary paths can be tiered safely. The matching shim
and daemon require `OBJECTVEIL/1` requests; native state, storage and configuration
have explicit format and namespace validation. Coordinate both components during
an upgrade, and follow [upgrading](../upgrading.md) for incompatible formats.

## Qualification checklist

Before supporting another application:

1. Define the complete managed file scope and an independent completeness
   oracle. Exclude databases, journals, credentials, temporary state and
   application metadata unless their semantics have separately been validated.
2. Verify the application uses supported interception paths. Direct syscalls,
   static binaries, memory mappings or operations outside the shim's supported
   hooks may need another adapter. Do not assume `LD_PRELOAD` covers them.
3. Prove that all writes coordinate with eviction. Exercise writes that pause
   with an open descriptor, rewrite a path in place, rename a temporary file,
   replace an existing file and run concurrently with reads and eviction.
4. Define deletion semantics explicitly. The Perforce integration translates
   managed archive deletion into remote authority/key operations. A different
   application's ordinary cleanup must not inadvertently request irreversible
   deletion of content it still needs.
5. Exercise path reuse, delete/re-add, identical content, shared packed/dedup
   objects, permission changes, missing keys and interrupted publication.
6. Test process crashes, full disks, backend/key-service outages, stale catalog
   reads and complete host loss. Preserve unpublished local writes and document
   the asynchronous recovery-point limit.
7. Supply application-specific backup, restore and cutover validation. Archive
   payloads alone do not restore Perforce databases, and the same principle
   applies to another application's metadata.

The existing multi-agent reviews and Perforce/cloud tests establish their
recorded scenarios only. A new adapter needs its own adversarial review and
failure tests before being described as supported.

## Path reuse

Perforce archive names can be reused after obliteration, and RCS archives change
in place. The implemented path epochs, poison chains and causal catalog must be
respected. An immutable-name assumption is not a qualification shortcut. Format
conversion must preserve revocation protections and authenticate existing keys.

See [obliterate/re-add epochs](obliterate-readd-epoch.md),
[the causal catalog](catalog-shard-crdt.md),
[deduplicated content](content-dedup.md),
[object-store requirements](object-storage-best-practices.md) and
[disaster recovery](../disaster-recovery.md).
