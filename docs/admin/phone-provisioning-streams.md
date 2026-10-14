# Provision streams (System + Customer)

Desk-phone config is built from **`#INCLUDE`** fragments. PBX3 ships **System** (package) streams and lets each tenant keep **Customer** fragments that survive package upgrade.

## System vs Customer

| Source | Where | Editable? |
|--------|--------|-----------|
| **System** | `/opt/pbx3/provisioning/streams/` on the home | Read-only in SPA (package owns them) |
| **Customer** | Tenant table `provision_stream` | Create / edit / delete in **Endpoints → Provision streams** |

Resolve order for each `#INCLUDE name`: **Customer** row for this tenant → else **System** file → else miss (omit lines; provision GET still succeeds).

## Authoring pattern

Prefer **additive** stacks. On vendors that are last-wins (Yealink, many Snom keys):

```text
#INCLUDE snom.Extension
#INCLUDE snom.udp
#INCLUDE site.ReceptionBLF
```

Put only the keys / BLF lines to **add or change** in the Customer fragment. Do **not** replace a System file under the same name (that mutes upgrades).

Closed XML (Poly) and some Fanvil sequences may need whole-stanza Customer bodies — the engine still only concatenates INCLUDE; authoring absorbs the oddity.

## SPA

1. Open **Provision streams**, pick the tenant.
2. Browse System (read-only) or **Copy to customer…** to learn / trim.
3. **New customer fragment** — prefer `site.…` names.
4. On the extension **Provision stream**, `#INCLUDE` System first, then your `site.…` names.

Deleting a Customer fragment that is still `#INCLUDE`’d returns **409** until you confirm force.

## Secrets

Prefer **`$…` symbolics** gated by extension **sndcreds** (Once / Always / No). The SPA soft-warns on lines that look like hardcoded `password=` / `secret=` values; it does not hard-block.

## Related

- [Desk phone RPS enrollment](phone-provisioning-rps.md)
- [Reseller full-config (M1)](phone-provisioning-m1.md) — when PBX3 does not own the HTTP stream at all
- [Extensions](extensions.md)
