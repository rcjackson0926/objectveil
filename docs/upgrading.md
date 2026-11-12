# ObjectVeil upgrades and format transitions

Use a complete, matching daemon, preload library and operator-tool release.
Version 7 uses configuration format `objectveil/1`, a non-nil `namespace_id`,
versioned storage identities and signed license envelope version 2. Do not
point it at an incompatible populated namespace or fabricate ownership markers.
A refused startup preserves the data; deleting state to bypass that refusal is
not an upgrade procedure.

## Fresh installation

Generate a UUID once (for example, `python3 -c 'import uuid; print(uuid.uuid4())'`).
Put it in `namespace_id` and set `format` to `objectveil/1` in the JSON config.
Use the same UUID on every writer, reader and migration configuration sharing
that data/key namespace. Keep distinct deployments on distinct namespaces.
The UUID is public identity, not a password. The nil UUID in templates is a
placeholder and is deliberately rejected.

Create the local archive directory under the application account. Before
starting the daemon, run:

```sh
objectveil-initialize /etc/objectveil/config.json
```

An empty archive root is the default. To use an existing **archive-only** root,
review its contents and use `--existing-archives`. This attests the tree contains
application archives, not databases, journals, secrets or foreign control state.
The initializer refuses symlinks, special nodes and incompatible ownership. It
does not copy, upload or delete archives. Its proof and state markers bind the
root to the configured namespace; preserve them with local recovery state.

A writer can initialize empty data/key namespaces. A read-only cache requires
existing matching remote identity markers and never initializes remote storage.
The primary and keystore must both belong to the same namespace, with their
correct distinct roles. A missing keystore beside populated data is an error,
not permission to generate replacement keys.

## Updating a compatible installation

1. Preserve configuration, signed licenses, complete local state, application
   metadata backups and independently recoverable archive coverage.
2. Stop or fence application writers, daemon background workers and failover
   automation. Replacing the on-disk preload library does not replace a library
   already mapped in a running process.
3. Verify release checksums and install the complete artifact set. Keep the
   prior set for rollback. Preserve `namespace_id`, all resolved cloud prefixes,
   `depot_path`, explicit state location, licenses and clock/trial anchors.
4. Initialize only a reviewed archive-only root if it has no native proof. Do
   not create markers by hand or adopt an unmarked populated state directory.
5. Restart the matching daemon and application under the archive-owner UID.
   Verify warm/cold reads and recoverability before authorizing eviction.

## From 7.0.0 to 7.0.1

7.0.1 belongs to the same format generation as 7.0.0. It converts nothing:
follow [Updating a compatible installation](#updating-a-compatible-installation).
Three things behave differently on the host afterwards:

- At its first start the daemon removes world access from an existing
  `manifest.redb`. New manifests are created owner-only.
- A configuration with an Azure backend that lacks `account_name`, or an S3
  backend with only one of `access_key` and `secret_key`, is now rejected when
  it is loaded. Such a configuration could not reach its backend before;
  correct it before upgrading.
- Removing the Debian package refuses while the service or a `p4d` is running,
  as installing and upgrading it already did, and no longer stops the service.

If `objectveil-verify` from 7.0.0 was ever run as root with Perforce checks
enabled, confirm that `/dev/null` on that host is still a character device
(`ls -l /dev/null` begins with `c`). That release let the Perforce client
replace it with a regular file; a reboot, or `rm -f /dev/null && mknod -m 666
/dev/null c 1 3` as root, restores it.

## Incompatible persisted formats

A format transition requires an explicit, separately verified conversion into
new native namespaces and state, while preserving the source. Contact
[rusty@rcjacksonconsulting.com](mailto:rusty@rcjacksonconsulting.com) for the
conversion procedure appropriate to the installed format. It must authenticate
source payloads and keys, preserve archive paths, tombstones and license/trial
identity, and cold-read verify the converted data before cutover. A finite copy
of a changing server is not an atomic snapshot or a deletion mirror.

Do not reset trials, alter signed claims, rewrite keys by string substitution,
remove poison records or discard the last recoverable copy. Conversion is not
performed by package installation, the initializer, or a service restart.
Keep source state and binaries fenced and recoverable until the new deployment
has passed recovery verification and the retention decision is explicit.
