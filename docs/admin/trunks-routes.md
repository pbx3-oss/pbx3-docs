# Trunks and outbound routes

## Trunks

Create / edit SIP trunks (ITSP or peer). Save → **Commit**.

Fleet design: **carriers usually live at the SBC / peering edge**, not as a fragile copy-per-node. Lab may still show instance trunks for solo trails.

### Solo inbound transform

On a **solo / direct** carrier trunk, set **Inbound transform** so national (or other) DID faces match `+E.164` inroutes before Ingress. Same `left:right` grammar as outbound transform (example UK: `0:+44`). Leave empty on fleet **Egress** — the SBC already emits `+`. Outbound transform stays separate (often the inverse face toward the ITSP).

See [Number wire](../fleet/number-dialect.md) and product **`NUMBER_WIRE_POLICY.md`** (solo inbound).

## Outbound routes

Route: `/routes`

1. Open route → dial patterns / COS / trunk selection.
2. Save → **Commit**.

Test with a known extension after Commit, not only from the SPA.

Fleet PSTN number formats (national vs +E.164 per carrier; node DNID transform vs CLID as-is; `+CC` by serving country): see [Number dialects](../fleet/number-dialect.md).
