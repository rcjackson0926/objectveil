# ObjectVeil licensing — operator guide

Upgrading an existing installation? Follow the
[ObjectVeil upgrade and format guide](upgrading.md) before changing
services, state directories, environment variables or cloud prefixes.


How to install a license, what the daemon does with it, what the
error messages mean, and what to send to support when something
goes wrong.

## TL;DR

1. Place your vendor-issued `.lic` file at `/etc/objectveil/license.lic` (or one of the alternate search paths below).
2. Restart the daemon. Look for `objectveil license loaded` in the operator log — it carries your `customer_id`, `capacity_tib`, and `signed_expires_at` / `effective_expires_at`.
3. If you see `running under the in-binary developer license`, you do not have a customer license installed and the daemon is using the built-in developer fallback. **Production deployments must override this.**

## License file shape

The `.lic` file is a single canonical-JSON document, signed by the vendor with Ed25519. You can `cat` it and read every field — that is intentional. The signed payload includes:

| Field | Meaning |
|---|---|
| `customer_id` | Stable customer slug. Appears in Prometheus labels, the startup banner, and the audit log. |
| `customer_name` | Human-readable customer name. |
| `license_id` | UUIDv4 unique to this issued license. Reference this in support tickets. |
| `tier` | One of `design_partner`, `annual`, `perpetual`, `ela`. |
| `capacity_tib` | Declared depot capacity ceiling. |
| `growth_band_pct` | Allowed overage before the hard ceiling kicks in. Defaults to 20. |
| `expires_at` | License expiration date (RFC 3339). |
| `maintenance_expires_at` | For perpetual licenses: when maintenance ends. |
| `features` | Allow-list of storage backends. |

The version 2 envelope binds product `objectveil`, envelope version, vendor key ID, algorithm and canonical claims under an ObjectVeil signature domain. Changing any signed value invalidates verification. JSON formatting alone is not entitlement identity. Verification happens at startup.

## Search paths (first match wins)

The daemon looks for a license in this order:

1. `--license <path>` command-line flag.
2. `OBJECTVEIL_LICENSE`.
3. `<config-dir>/license.lic`, beside the selected config JSON.
4. `/etc/objectveil/license.lic`.
5. The built-in developer license when no customer license was selected; this
   fallback is not intended for production.

An explicit missing path is a hard error. Unsafe, invalid or incompatible files
are refused. Customer license files for version 7 require a vendor-signed ObjectVeil version 2 envelope;
older envelope formats require verified conversion, not editing signed JSON.
Preserve license UUIDs, entitlement dates, customer identity, authenticated trial
first-launch time and clock highwater during a format transition. Conversion
must not reset a trial. See [upgrading](upgrading.md).

## What enforcement looks like

ObjectVeil applies staged admission/upload restrictions. A running daemon's
license policy continues to permit reads in the states below, subject to normal
backend, key, capacity and operational availability; it does not guarantee an
outage-free deployment. Startup can refuse a license beyond its grace period.

### Capacity axis

| State | Meaning | What happens |
|---|---|---|
| Green | < 80% of `capacity_tib` | Normal operation. |
| Warn | 80–100% of `capacity_tib` | Tracing `warn` log at the threshold cross; no functional change. |
| Grace | 100–120% (within `growth_band_pct`) | Tracing `error` log; serves everything. Negotiate before crossing the hard ceiling. |
| Refused | ≥ 120% | New paths wait for admission as durable Dirty entries; their local bytes remain queued. Already-admitted entries continue to upload and serve. Reads continue. |

The active state is exported as `objectveil_license_capacity_state` (0/1/2/3) and the sampled depot bytes as `objectveil_license_managed_bytes`. The sampler runs once per minute.

### Expiry axis

| State | Meaning | What happens |
|---|---|---|
| Healthy | More than 30 days before `expires_at` | Normal operation. |
| NearingExpiry | Within 30 days | Tracing `warn` log on transition. Renew before `expires_at` to avoid the read-only grace window. |
| Expired | Past `expires_at`, within 30-day grace | Tracing `error` log. New admissions and all cold-tier uploads pause; observed writes remain durably queued for renewal. Reads continue. |
| BeyondGrace | 30+ days past `expires_at` | A running daemon stays running; the **next restart fails to load** the expired license. Rotate to a renewed license before the next restart. |

Exported as `objectveil_license_expiry_state` (0/1/2/3) and `objectveil_license_expires_at_unix`.

## Observability

`objectveil_license_info` is a separate constant gauge carrying deployment identity; it does not add these labels to every other metric:

```
objectveil_license_info{customer_id="acme-corp",customer_name="Acme Corp",
                     license_id="...",tier="annual",source="signed",
                     vendor_key_id="v1"} 1
```

Other metric families do not inherently carry `customer_id`. Correlate the identity gauge through scrape identity such as `instance` and `job`, or explicitly provision consistent operator labels before building fleet attribution queries.

## Cloud-side watermarks

The same identity propagates into every cold-tier request the daemon makes to S3/Azure/GCS (all three go through the single `object_store` client — there is no per-cloud SDK), in two forms:

1. **HTTP `User-Agent`** — suffix `objectveil/<version> (customer=<customer_id>;source=<source>)`, set on the `object_store` HTTP client (`ClientConfigKey::UserAgent`) so every S3/Azure/GCS request carries it. Visible in:
   - AWS CloudTrail (`requestParameters.userAgent`)
   - Azure Activity Log
   - GCS: carried on every request the same way; the service-account principal is also recorded in Cloud Audit Logs independently.
2. **Object-level metadata** on every cold-tier write. Keys use **underscores, not hyphens** (Azure Blob rejects hyphenated metadata names with HTTP 400; underscores are the portable form across all three backends):
   The existing internal field names carry customer ID, license ID, source
   (`signed` / `developer-build`), tier, and a persistent deployment UUID.
   The UUID is minted on first launch and cached at
   `<state_dir>/license/daemon-instance.uuid`, so it stays stable across
   restarts and distinguishes multiple instances of the same license.
   The metadata fields are `objectveil_customer_id`, `objectveil_license_id`,
   `objectveil_daemon_instance`, `objectveil_source` and `objectveil_tier`.
   
   Stored as `x-amz-meta-*` on S3, `x-ms-meta-*` blob metadata on Azure, `x-goog-meta-*` custom metadata on GCS. Survives long after request logs roll over.

The daemon-instance UUID is logged at startup (`daemon_instance_id` field on the `license watermark active` line). Include it in support tickets alongside `license_id`.

Migration writer, verification and download jobs use the original configured
installation's trial/clock anchors and UUID. Private job directories do not
establish new trial windows or deployment identities. Canonical anchor loads
and reloads take a short nonblocking cross-process lock; retry a busy load after
the other operation finishes. Migration audit logs remain private to the job
rather than competing with a running daemon's audit writer. Preserve existing
license state when changing paths; see [migration](migration.md).

## Audit log

`<state_dir>/license-audit.log` is an NDJSON file. With a signed customer license, license events (load, capacity transition, expiry transition, integrity-check result) form a BLAKE3 keyed-MAC chain using the issued `customer_secret`. Developer mode is explicitly unchained and is not equivalent authenticated audit evidence. A daemon restarting against an existing log continues the same chain from its terminal `(seq, hmac)`; a companion anchor file (`license-audit.anchor.json`) records that terminal pair after every write so truncation of the log tail is detectable on the next startup.

```
{"product":"objectveil","v":2,"seq":42,"ts":"2026-05-18T10:23:45.123Z","prev_hmac":"...","hmac":"...","event":{"event":"license_loaded","customer_id":"acme-corp",...}}
```

When opening a support ticket, attach the audit log alongside the daemon log. The vendor can verify the chain is intact end-to-end.

## Integrity check

At every startup, the daemon BLAKE3-hashes every executable (`PF_X`) `PT_LOAD` segment of its own in-memory image — walked via `dl_iterate_phdr`, in `p_vaddr` order — and compares against a value baked into the binary by the vendor's `integrity-stamp` post-link tool. (Hashing the whole executable-segment set, not just a section named `.text`, covers code an LTO/PGO build may have split into `.text.hot` / `.text.unlikely`.) Possible outcomes:

* `Passed` — info log. Binary is intact.
* `Failed` — error log + audit-log entry. The executable segments do not match the stamped value. Causes:
  * In-place patch / debugger breakpoint / hot-patch ABI
  * Build pipeline misconfiguration (binary wasn't stamped)
  * Bit-rot on storage / accidental partial file
  
  This is a **hard fail** by default: the daemon **refuses to start** (`exit 1`). Stop and investigate the artifact mismatch; do not use a development override to bypass a failed release integrity check. **`INTEGRITY_FAIL` in your support diagnostics voids the license per the support agreement**.
* `Unstamped` — the executable has no vendor integrity seal. This can occur in a development build; it is not acceptable release-artifact evidence. Use the matched, checksum-verified release and contact support if a distributed artifact reports this result.
* `Skipped` — warn log. `dl_iterate_phdr` enumerated no executable LOAD segments, or the platform is not Linux (the segment walk is Linux-only; macOS is not implemented). Daemon continues.

## Common error messages

### `license load failed: license file not found at any search path`

Check the selected license path, its ownership and permissions, and the version 2 license requirement. Do not delete authenticated clock/trial state or substitute development credentials to bypass a refusal. Contact support if the supplied license cannot be loaded.

### `license load failed: license signature did not verify`

The `.lic` file's signature does not match any of the vendor public keys embedded in this daemon binary. Causes:

* A signed claim, product, envelope version, key ID, algorithm or signature has
  changed. Signatures cover canonical claims, so changing JSON whitespace or
  line endings alone does not invalidate an otherwise valid license. A malformed
  encoding or BOM may instead prevent parsing.
* The file was issued by a different vendor key than the binary trusts (occurs during key rotation).
* The file is for a different product.

Re-download the file from the vendor and try again.

### `license load failed: license signature references unknown vendor_key_id`

The license names a `vendor_key_id` (e.g. `v1`) that this daemon binary doesn't carry. Almost always means you're running an older daemon than the license was issued for. Upgrade the daemon.

### `license expired ... more than 30 days past expires_at`

Self-explanatory. Issue a renewed license.

### `write retained locally and queued pending license admission`

The license is expired or the depot has reached its capacity ceiling. New files remain tracked as Dirty and cannot be evicted. Renew the license or resolve the capacity limit; the persistent queue retries without requiring another write notification. Already-admitted files may still upload while capacity is exceeded. Uploads paused by expiry require renewal.

Queued bytes are still local-only until successful publication. Loss of the writer and its disks during this window can lose those writes; license refusal does not provide a separate backup.

## Support flow

When opening a ticket, attach (in this order):

1. The exact `license_id` from the startup banner.
2. The last 5 minutes of operator log around the failure.
3. The `license-audit.log` (the full file is fine; it persists and grows across restarts, and — when the operator provisions `chattr +a` — is verified filesystem-append-only at startup).
4. The installed artifact version, its release-checksum result, and the startup integrity-check result. The vendor integrity-stamp utility is not included in the public nine-artifact release; request additional verification tooling from support if needed.

Vendor-side verification requires the issued `customer_secret` and the
corresponding authenticated anchor as well as the log. A `customer_id` alone
cannot verify the keyed chain. Preserve the anchor when collecting diagnostics;
coordinate any transfer of license secrets through the vendor's secure support
process.
