# Fleet overview

Many interchangeable **instances**. A **tenant** lives on one instance at a time and can move without reconfiguring phones.

| Piece | Job |
|-------|-----|
| **SBC** | Single SIP front door; routes by tenant domain |
| **Instances** | Asterisk + API + tenant data |
| **Org S3** | Catalog, backups, recordings — not on the call path |
| **Gatekeeper** | Fleet control plane (`control.pbx3.com` in lab) |
| **SPA Fleet mode** | Operator UI for catalog / jobs / DIDs — shared at **[app.pbx3.com](https://app.pbx3.com)** (not per-node / per-builder hosting) |

Calls keep working if Gatekeeper or S3 is down.

## Solo vs fleet

| Solo | Fleet |
|------|-------|
| One node; S3 optional | Shared bucket + catalog + usually SBC |
| Instance Sanctum only | + Gatekeeper fleet token |

See also the schematic on [What is PBX3?](../getting-started/what-is-pbx3.md).

## Lab map

Phones → **sbc.pbx3.com** → **08jzwn** / **bzy54n**; bucket **`08jzwn-pbx3`**; Gatekeeper **control.pbx3.com**.

## Cross-tenant short dial

Sister sites short-dial via **[Site Groups](site-groups.md)** (Fleet UI): destination routing prefix + membership mesh. Hand-entered dial prefixes are lab / break-glass only.

## DIDs

Carrier numbers: allocate in **[Fleet → DIDs](dids.md)** (hop 1 → home); route behaviour on the instance **[Inbound routes](../admin/inbound-routes.md)** (hop 2). Do not author fleet inbound delivery on the SBC Number routes panel.

## Instance lifecycle

| Task | Page |
|------|------|
| New home (Install → onboard → edge) | [Commission](commission-instance.md) (Install first) |
| Join a healthy node (adopt only) | [Onboard](onboard-instance.md) or Commission Step 1 |
| Retire a home (tenants → Decom → SBC → terminate) | [Decommission a fleet instance](decommission-instance.md) |
| Same KSUID after EC2 loss | [Rebuild from S3](rebuild-from-s3.md) |
| Wipe one site | [Tenant delete](tenant-delete.md) |
| Home a built tenant from a compatible DB | [Ingest a compatible tenant DB](ingest-compatible-db.md) |
| Recordings → S3 offload on/off (ops `.env` today) | [Recordings S3 offload](recordings-s3-offload.md) |
