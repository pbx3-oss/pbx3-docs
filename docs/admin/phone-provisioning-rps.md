# Desk phone RPS enrollment

Point vendor **Redirect / Provisioning Server (RPS)** at PBX3 so the handset downloads its config over HTTPS. SIP registration is separate (tenant FQDN via the SBC).

## Prerequisites

- Extension exists with a **MAC**, provision stream, and credentials.
- **Fleet:** MAC is claimed in the catalog index and the SBC has synced `provision-mac.map`.
- Phone can reach **TCP 41363** on the provision hostname (solo: instance; fleet: edge VIP).

## Which URL to enroll

| Mode | RPS / setting-server base |
|------|---------------------------|
| **Solo** | `https://{instance-fqdn}:41363/provisioning/…` |
| **Fleet** | `https://provision.{apex}:41363/provisioning/…` (e.g. `provision.pbx3.com`) |

On fleet, **never** enroll the home instance FQDN — the edge terminates TLS and proxies by MAC so phones do not learn home URLs.

## Vendor URL forms

Normalize MAC to **12 hex digits** (no colons) unless the vendor portal fills `{mac}` for you.

### Yealink

Path form (preferred):

```text
https://provision.pbx3.com:41363/provisioning/{mac}.cfg
```

Example: `https://provision.pbx3.com:41363/provisioning/249ad89b435b.cfg`

### Snom

Query form (SRAPS / setting server):

```text
https://provision.pbx3.com:41363/provisioning?mac={mac}
```

Example: `https://provision.pbx3.com:41363/provisioning?mac=000413BE48C0`

### Gigaset (note)

Often **MAC + PIN** in the vendor portal rather than a full URL — see your Gigaset Redirect docs. Deep stream coverage is second-wave.

## Solo → fleet flip (manual, rare)

If phones were enrolled on the **instance** URL and the deployment later joins a fleet:

1. Confirm `provision.{apex}` DNS + TLS and MAC map sync on the SBC.
2. Re-point each enrolled MAC’s RPS target from `https://{old-instance}:41363/…` to `https://provision.{apex}:41363/…` (same path or `?mac=` form).
3. Trigger a re-provision on the handset.

There is no bulk migrator yet; this is a one-time operator change. After fleet is live, **tenant rehomes do not require another RPS edit** — the MAC map follows the new home.

## Related

- Extension Save → Commit and SPA provision URL / Reset Once — [Extensions](extensions.md)
- Tenant move (SIP/catalog; RPS stays on `provision.{apex}`) — [Tenant move](../fleet/tenant-move.md)
- Reseller / vendor delivers full config (no PBX3 HTTP) — [Reseller full-config (M1)](phone-provisioning-m1.md)
