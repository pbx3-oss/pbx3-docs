# Tenant move

Move a tenant from one fleet home to another **without** re-pointing desk RPS when phones use `provision.{apex}`.

High-level order (Fleet → **Tenants** → Move / **Jobs** wraps this):

1. Prep destination capacity / trunk mapping (trunks **do not** move).
2. Export on source → import on destination → **Commit** → test on dest **before** cutover completes.
3. Cutover (fleet SBC): catalog domain `setid` **and** DID hop-1 projection → destination (automated in the job).
4. Cert **Sync** on both nodes as SANs change (LE sync in the job is best-effort; SPA Sync if it skips).
5. **DID hop-1** follows cutover automatically (Gatekeeper projects `fleet=did` rules to the destination dispatcher setid). Confirm with Fleet → DIDs reconcile if you want a second look — see [DIDs](dids.md).
6. **Drain, then wipe source** — see dual-copy below.

## Dual-copy until wipe (important)

After cutover / catalog repoint and **before** wipe:

| On destination | On source |
|----------------|-----------|
| Live home of record (catalog + SIP setid) | Tenant rows still present until you wipe |
| Phones should REGISTER here | Orphan risk if you skip wipe (SPA may show raw shortuid) |

The job sits at **`awaiting_cleanup`**. That is intentional:

- Wait for phones to re-register / soak on dest.
- You may leave the job page; reopen via Fleet → **Jobs** → **Open**.
- Then **Wipe tenant on source** (irreversible cascade + Commit). Do **not** start a second Move for the same wipe.

**Rollback boundary:** anything before wipe is recoverable (flip setid / abort). After wipe, restore = re-import from the staging zip (retained N days).

## Phone provisioning on move

| Path | What to do |
|------|------------|
| Fleet RPS → **`provision.{apex}:41363`** (M3) | **Do not** change RPS. MAC index rewrite updates the edge map to the destination home. Next provision GET follows automatically. |
| Manual / solo instance URL | Rare; re-point RPS or phone URL if still enrolled at the old home. |
| Reseller full-config (**M1**) | Outside PBX3 MAC map — follow the reseller platform. |

Customer **Provision streams** (`provision_stream`) travel in the tenant miniDB export/import with other cluster-scoped tables. System stock streams stay on each home’s package.

Edge **Provision access** lockdown (optional UFW on `:41363`) is **edge-global** — site CIDRs do not change on tenant move.

See [Desk phone RPS enrollment](../admin/phone-provisioning-rps.md) · [Provision streams](../admin/phone-provisioning-streams.md) · [Restrict provision HTTPS](../admin/phone-provisioning-access.md).

## Desk BLF / PARK after rename or rehome (lab gotcha)

**Not a move-job defect** — usually bad lab housekeeping: a handset BLF that was never re-programmed after the tenant FQDN changed.

Snom (and similar) often expand a short BLF target at **key-program time**. You type `901` for PARK; the phone stores an absolute URI such as `sip:901@{tenant-then}.pbx3.com`. REGISTER / line identity can follow the new home; that BLF URI does **not**.

**Symptom:** phone log shows registration / transport timeout to the SBC VIP; line UI goes unregistered. SBC still has a healthy `REGISTER` for the extension at the **current** tenant FQDN. OpenSIPS logs `SUBSCRIBE` to the **old** FQDN with `Door-knock blocked: domain=… (not found)` and no SIP reply — the handset then promotes the subscription timeout into a fake line failure.

**Lab example (2026-10-06):** Snom PARK still `sip:901@nqybwn.pbx3.com` after Aelintra lived as `hf3zzv`; Call-ID on the “unregistered” event was that SUBSCRIBE, while `3cg94b@hf3zzv` REGISTER kept returning 200.

**Fix:** clear and re-enter the PARK/BLF target on the phone so it re-expands against the current registrar domain (or fix the absolute URI in the provision stream / site fragment if you manage keys that way).

## CLI (lab / break-glass)

```bash
# source
sudo -u www-data php artisan tenant:export {tenant}
# dest
sudo -u www-data php artisan tenant:import /opt/pbx3/bkup/pbx3tenant.{shortuid}.*.zip
# catalog
./move-tenant.sh --tenant-shortuid {s} --instance-id {DEST_KSUID} --fqdn {tenant.fqdn}
```

Prefer the Fleet Move job when available — it owns cutover, catalog, and the wipe gate.

Lab worked example: **08jzwn** → **bzy54n** (`hf3zzv` rehome soak 2026-10-02: map follow + phones REGISTER/calls + source wipe).

## Related design

- Product design: **`TENANT_MOBILITY_FLEET_CONSOLE_DESIGN.md`** (job state machine, `awaiting_cleanup`, rollback).
- Orphan rows if wipe skipped: design risk **1b** — always wipe after verify.
