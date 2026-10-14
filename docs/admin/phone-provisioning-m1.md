# Reseller / vendor full-config provisioning (M1)

PBX3 does **not** require phones to download config from our HTTP provisioner. If a vendor cloud, OEM partner, or reseller already delivers the **complete** phone config (SIP identity, registrar, codecs, keys), that path stays valid.

Call this **M1**: we own **SIP / Commit / PJSIP** on the home; someone else owns the **final HTTP config stream**.

## When M1 fits

- The MSP or customer already lives on a reseller or vendor management cloud end-to-end.
- Handsets never need to hit `https://provision.{apex}:41363/…` or the solo instance provision URL.
- You still create extensions (MAC optional), set passwords, and **Commit** so Asterisk can accept REGISTER.

## What PBX3 still does

| Layer | M1 role |
|-------|---------|
| Extension + SIP secret | Home of record — same as always |
| Commit / GenAst / PJSIP | Required for REGISTER to succeed |
| Fleet SBC | Still the SIP edge for fleet desks |
| Our provision HTTP (`:41363`) | **Unused** for those MACs |

Do **not** enroll those MACs in vendor RPS with a PBX3 provision URL. Leave RPS / YMCS / SRAPS / reseller templates pointed at the reseller’s own config host.

## Coexistence with PBX3 provision (M3)

M1 and PBX3-owned provision (**M3**) can mix on the same instance or fleet:

- **M3 phones** — RPS target = solo instance `:41363` or fleet `https://provision.{apex}:41363/…` (see [Desk phone RPS enrollment](phone-provisioning-rps.md)).
- **M1 phones** — RPS / management cloud delivers full config elsewhere; PBX3 never sees a provision GET for that MAC.

Rules of thumb:

1. One MAC should not be enrolled for **both** a reseller full-config URL and a PBX3 provision URL.
2. Tenant move / rehome: M3 fleet phones with `provision.{apex}` need no RPS edit. M1 phones follow whatever the reseller platform already does — PBX3 MAC map does not apply.
3. SPA **Last provisioned** / **Reset Once** only matter for MACs that fetch from us.

## What we deliberately do not build

- A product that replaces vendor/reseller **redirect / discovery**.
- A requirement that every desk phone use our HTTP provisioner.

Thin MAC export for someone else’s portal (**M2**) may come later; it is not required to stay on M1.

## Related

- [Desk phone RPS enrollment](phone-provisioning-rps.md) — when PBX3 owns the final config stream
- [Extensions](extensions.md) — SIP credentials and Commit
- [Tenant move](../fleet/tenant-move.md) — fleet mobility (M3 + `provision.{apex}`)
