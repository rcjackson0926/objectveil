# Red Hat Enterprise Linux installation

Use `objectveil-7.0.1-1.el9.x86_64.rpm` for RHEL 9 on Intel/AMD 64-bit.
This native build uses glibc 2.34. The Debian/Ubuntu package and generic Linux
tarball are separate builds requiring glibc 2.39; do not use those on RHEL 9.
RPM resolves the generated ELF dependencies, including glibc and libgcc,
plus systemd, procps-ng and ca-certificates for public-cloud HTTPS trust. No source implementation, license, credentials
or vendor signing/stamping utility is included.

## Installation

Download the RPM and `SHA256SUMS` from the official release into an empty
directory, verify each downloaded asset, and complete the coordinated stop
in [release installation instructions](https://github.com/rcjackson0926/objectveil/releases/download/v7.0.1/INSTALL.txt) before installing:

```sh
sha256sum --ignore-missing -c SHA256SUMS
sudo dnf install ./objectveil-7.0.1-1.el9.x86_64.rpm
rpm -q objectveil
rpm -V objectveil
```

Both a first install and an upgrade refuse while any p4d or ObjectVeil daemon
is running, or the service is starting/stopping. Keep automatic restarts fenced
through the complete handoff. The package never stops writers, starts/enables
the daemon, creates a Perforce account, or initializes configuration and state.
The unit expects `perforce:perforce`; use an override for your archive owner.

Executables live in `/usr/bin`, the matching preload library is
`/usr/lib/objectveil/libobjectveil_shim.so`, the unit is in
`/usr/lib/systemd/system`, and guides are in `/usr/share/doc/objectveil`.
Follow INSTALL.txt to create a native configuration with a fresh deployment
UUID, supply a valid license, explicitly initialize the reviewed archive-only
root, and start the daemon before restarting every writer with its matching shim.
Do not mix RPM and Debian/tar runtime components even when their version matches.

## SELinux and host verification

Keep SELinux enforcing. Archive paths, custom state/socket paths, the Perforce
service domain and cloud access depend on your installation. Review denials
with your administrator and configure appropriate persistent file contexts and
a narrowly scoped local policy when required. Do not generate/approve broad
allow rules blindly or disable SELinux to make a failing deployment start.
This package does not install a universal SELinux policy or relabel archives.

Validate the real host's fanotify support and service capability, archive-owner
permissions, license and a real cold read under the application UID
before enabling eviction. Container tests exercise RHEL user space, not the
RHEL kernel, host systemd or your enforcing SELinux policy.

## Live migration without replacing an installed shim

A running server can use the migration command from a separate extraction.
Verify the RPM first. As an unprivileged archive owner, create a new empty
private directory and extract the trusted payload there (requires rpm2cpio/cpio):

```sh
mkdir -m 700 objectveil-rhel9-migration
cd objectveil-rhel9-migration
rpm2cpio ../objectveil-7.0.1-1.el9.x86_64.rpm | cpio -idm --no-absolute-filenames
./usr/bin/objectveil-migrate --help
```

Run only this copy of the migration tool following [migration.md](migration.md).
Do not preload its shim or activate a cache writer until the coordinated
handoff. Busy files are skipped and reported; resume later. The capture is per
file, not an atomic snapshot or a deletion mirror. Uploads use the cache's
packing, encryption, deduplication and parallel multipart pipeline.

## Upgrade and removal

Fence automatic restarts, stop all p4d/writers and ObjectVeil, then install
with `dnf upgrade ./objectveil-7.0.1-1.el9.x86_64.rpm`. Follow the same matching
shim restart and recovery checks as a fresh install. Consult [upgrading.md](upgrading.md)
for format/identity preservation and rollback requirements.

For removal, complete the same coordinated stop, then:

```sh
sudo dnf remove objectveil
```

Removal refuses if writers/daemon are active. Package scripts never delete
operator configuration, signed licenses, archives, keys, local state or trial
anchors. Only the shipped example is `%config(noreplace)`; RPM may preserve
local edits as `.rpmsave` on removal or use `.rpmnew` during updates. The real
`/etc/objectveil/config.json` is never supplied or overwritten by the package.

The RPM is not GPG-signed. Download it and its checksum from the expected
release and verify SHA256 before using `dnf`. Checksums and the runtime's
integrity seals are not a detached package signature or Red Hat certification.
