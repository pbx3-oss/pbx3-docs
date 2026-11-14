# Provisioning security (discussion)

Desk HTTP provisioning is an **assist**, not the call plane. This page records the security tradeoffs and a **solo** product decision: leave the singleton provisioner as shipped, and put **vendor certificate sourcing** on the operator.

Related how-tos: [RPS enrollment](phone-provisioning-rps.md) · [streams](phone-provisioning-streams.md) · [edge access allowlist](phone-provisioning-access.md).

## What the phone is doing

1. Discovers a provision URL (vendor/reseller **RPS**, DHCP opt66, or manual entry).
2. **HTTPS GET/POST** keyed by **MAC** (and sometimes a common vendor descriptor).
3. Applies the expanded stream (SIP user, password when `sndcreds` allows, registrar/proxy, softkeys, …).
4. **REGISTER**s for calls — separate from the provision hop.

Anyone who can complete step 2 for a known MAC can see whatever that GET returns (including secrets when `sndcreds` is `Always` / unspent `Once`). MAC is a **locator**, not strong authZ by itself.

## Two different certificate problems

Operators often collapse these; they are not the same:

| Problem | Who presents | Who verifies | Typical pain |
|---------|--------------|--------------|--------------|
| **Server TLS** | PBX / edge provision vhost | Phone trusts the **server** chain | Snakeoil / private CA → phone rejects HTTPS unless you install a trust root on the handset or disable cert checks (vendor-dependent) |
| **Client mTLS** | Phone’s **factory / vendor client cert** | Provision nginx verifies against a **vendor client CA bundle** (`3pcerts`-style) | Without the manufacturer CA on the server, you cannot require “only a real Yealink/Snom/…” — bare curl with a MAC still works |

**Decision hinge:** both depend on materials vendors do **not** ship inside PBX3. The operator (or her MSP) must obtain **vendor-supplied root / client CAs** through partner portals, NDAs, or regional packs. PBX3 will not pretend a public “download all phone CAs” button exists.

**Without those certs, local provisioning will always have rough edges** — TLS trust failures on the phone, or an open GET-by-MAC listener that cannot honestly claim mTLS.

## Solo (singleton) — locked stance (2026-11-14)

For **solo / direct** installs (no fleet SBC provision edge):

- **Leave provisioning as shipped.** No product requirement to turn on client mTLS on the home `:41363` listener.
- LAN / lab GET-by-MAC remains acceptable for private nets (prefer **`Once`** for passwords; reset provision state after factory reset).
- Harden with **network controls** (VLAN, firewall to `:41363`, no internet exposure of the provision port) rather than inventing a half-mTLS story without CAs.
- Server HTTPS still matters: prefer a real LE cert on the instance when phones are picky; snakeoil needs phone-side trust or vendor “ignore cert” for soak only.

Rationale: solo is Rule-6 try-it / office LAN. Shipping a mandatory vendor-CA pack in the product is blocked by licensing and inventory churn; forcing mTLS without CAs only creates false confidence.

## Fleet edge — different bar

Cloud / fleet phones hit **`https://provision.{apex}:41363`**. There the design already prefers:

- Optional **vendor client-cert verify** on the edge (lab-proven for some brands).
- Optional **site egress allowlist** on `:41363` ([Restrict provision HTTPS](phone-provisioning-access.md)).
- Edge → home over HTTP from SBC IPs only; secrets stay on the home.

Even on fleet, **missing a brand’s CA** means that brand cannot join a strict mTLS posture — use Once + network controls, or enroll via a vendor cloud that already holds the PKI, rather than “optional mTLS that accepts everything.”

## Operator responsibilities (short)

| You own | PBX3 owns |
|---------|-----------|
| Sourcing vendor root / client CA material for the brands you deploy | Home expand, streams, `sndcreds`, audit masking, solo HTTPS listener |
| Installing phone trust for non-public server certs when needed | Fleet edge proxy + optional mTLS/allowlist **when CAs and ops tip them** |
| Not exposing solo `:41363` to the open internet without intent | Documenting the model (this page + product requirements) |

## Practical mitigations without full mTLS

- Prefer **`Once`** (or `No` after enroll) so daily re-polls do not re-emit SIP passwords.
- After handset factory reset, **reset provision state** on the extension so `Once` secrets send again.
- Keep provision on a **management / voice VLAN**; block guest Wi‑Fi to `:41363`.
- Fleet: enable [Provision access](phone-provisioning-access.md) when site egress CIDRs are stable.
- Treat browser “open the provision URL” as an admin debug aid on a trusted LAN — same discomfort as viewing the SIP password in the SPA.

## Summary

Solo stays open-enough for LAN assist. Strong remote harden needs **vendor CAs the operator must supply**. PBX3 documents the issues; it does not ship a complete third-party CA catalogue or claim mTLS where the CA is missing.
