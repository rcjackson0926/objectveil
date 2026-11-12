# ObjectVeil release changes

## 7.0.1

A maintenance release in the same native format generation as 7.0.0. It changes
no configuration, catalog, key or manifest format and needs no conversion.

- Preserve shared-content GC wakeups when an obliterate or ingest completes
  during a pass, and track catalog publications through completion so a
  publication already underway cannot escape the pass's activity watch.
- Reject contradictory deduplication descriptors in remote catalog deltas and
  snapshots before they can establish restore, compaction or GC authority.
- Normalize cloud namespace prefixes consistently for uploads, reads and
  paginated listings. Leading or trailing slashes no longer produce listed
  keys that point at a different object on subsequent operations; listings
  outside the configured namespace are rejected.
- Prove shared-content garbage collection once per pass. Tier-2 GC now reads
  the remote catalog and the keystore once for a whole batch of unreferenced
  records, and does so without pausing uploads or cold restores. It previously
  repeated both reads for every record while holding the writer drain, and
  could not finish on a keystore of more than a few thousand keys. Blocked
  records are retried with back-off and no longer starve the others, what
  blocked a record is remembered, and each pass logs why it retained a record
  and a summary of what it did.
- Create the manifest database owner-only and remove world access from an
  existing one. In the packaged layout the state directory is searchable by
  every local account, which exposed tracked depot paths and content hashes.
- Stop the verifier from pointing the Perforce client's enviro file at
  `/dev/null`. The client rewrites that file by renaming a new one over it, so
  `objectveil-verify` run as root replaced the host's null device with a
  regular file. The client is now given a path that cannot exist.
- Reject an Azure backend without `account_name`, and an S3 backend with only
  one of `access_key` and `secret_key`, when the configuration is loaded.
- Refuse Debian package removal while the service or a Perforce server is
  running, as the RPM already did; removal no longer stops the service. The
  Debian package now depends on `ca-certificates`.
- Reach the EC2 metadata service directly for the cross-region warning, whatever
  proxy the environment configures.
- Correct the example configurations, which a test now loads through the real
  parser. The packaged example manages the Perforce depots directory.
- Bring the operating guides in line with the implementation: credential
  sources, validation rules, defaults, hot-reloadable fields, metrics, the
  built-in developer license and post-expiry behavior. Remove build-workspace
  instructions and the socket readiness step that no shipped tool can perform.
- Update `rustix` past RUSTSEC-2026-0486, and stop declaring the proprietary
  workspace crates as MIT-licensed.

## 7.0.0

- Require explicit native configuration and namespace identities, paired remote
  role markers, versioned protocol envelopes and authenticated catalog format.
- Add `objectveil-initialize` for explicit local archive-root ownership.
- Bind license signatures to the product, envelope and claims; preserve
  entitlement and authenticated trial identity through verified conversion.
- Treat object keys as opaque on NFS to prevent archive/control path collisions.
- Use the cache storage paths for migration packing, encryption, deduplication,
  multipart uploads and independent cold-read verification.

- Require complete catalog authority before eviction or cold restoration; retain
  recoverable payloads when deletion safety cannot be proved.
- Bound aggregate packing memory and startup recovery buffers, and preserve
  source evidence when backend metadata or state files are invalid.
- Preserve installation licensing anchors and identity across migration jobs.

Release acceptance is recorded separately for the exact source commit and
artifact hashes. Check the published validation receipt and checksums; a source
snapshot alone is not release acceptance.
See [upgrading](docs/upgrading.md) before changing persisted formats.
