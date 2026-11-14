# Provisioning security (discussion)

Desk HTTP provisioning is an **assist**, not the call plane. Calls work after SIP REGISTER even if no phone ever hits `:41363`. This page is the public discussion of **why** that assist is hard to lock down, what PBX3 will and will not ship, and the **solo** product decision of 2026-11-14.

How-tos: [RPS enrollment](phone-provisioning-rps.md) · [streams](phone-provisioning-streams.md) · [edge access allowlist](phone-provisioning-access.md) · [reseller full-config (M1)](phone-provisioning-m1.md).

---

## Decision (2026-11-14)

| Mode | Stance |
|------|--------|
| **Solo / singleton** | **Leave provisioning as shipped.** No product-forced client mTLS on the home `:41363` listener. Harden with LAN/VLAN/firewall + `sndcreds` (`Once` preferred). |
| **Fleet edge** | Optional vendor **client-cert verify** + optional [site egress allowlist](phone-provisioning-access.md) on `provision.{apex}:41363` — when the operator has CAs and turns them on. |
| **Vendor CAs** | **The operator owns sourcing.** PBX3 does not ship a complete OEM root/client CA catalogue. Without those materials, local TLS and honest mTLS stay rough. |

Rationale in one line: shipping mandatory phone CAs is blocked by partner licensing and inventory churn; claiming mTLS without them is false confidence.

---

## What the phone is doing

```text
Discover URL (RPS / DHCP opt66 / manual)
  → HTTPS GET/POST … keyed by MAC (± vendor “common” file)
  → apply expanded stream (SIP user, password if sndcreds allows, registrar/proxy, …)
  → SIP REGISTER (solo: instance; fleet: SBC)   ← separate hop
```

| Fact | Consequence |
|------|-------------|
| Request is usually keyed by **MAC** | MAC is a **locator**, not strong authorization |
| Response can include **SIP password** | Anyone who can complete the GET for that MAC can read secrets still gated as `Always` or unspent `Once` |
| Provision ≠ REGISTER | Closing provision does not stop calls; opening provision does not authenticate a caller |

That is why a LAN browser opening `https://…:41363/…?mac=…` feels uncomfortable: it works by design on an open GET-by-MAC listener, the same way the SPA can show a SIP password to a logged-in admin.

---

## Two different certificate problems

Operators often collapse these; they are not the same.

### 1. Server TLS (phone trusts the PBX)

| | |
|--|--|
| **Who presents** | Solo home or fleet edge provision vhost |
| **Who verifies** | The phone (OEM trust store ± admin-installed roots) |
| **Happy path** | Public Let’s Encrypt (or other public CA) on the provision hostname |
| **Pain** | Snakeoil / private CA → many phones **reject** HTTPS unless you install a trust root on the handset or disable cert checks (vendor-dependent, often buried) |

Solo lab often hits this first: API snakeoil for the SPA is one story; desk phones are pickier about the **provision** URL.

### 2. Client mTLS (PBX trusts the phone)

| | |
|--|--|
| **Who presents** | Phone’s **factory / vendor client certificate** |
| **Who verifies** | Provision nginx against a **vendor client CA bundle** (classic `3pcerts` / `SSLVerifyClient require`) |
| **Happy path** | Request must chain to an allowed manufacturer CA → random laptop curl without a phone cert fails |
| **Pain** | Without the manufacturer CA on the **server**, you cannot require “only a real Yealink/Snom/…” |

**mTLS is not a panacea.** It proves a **vendor/family** client cert, not “this exact MAC is authorized.” Pair it with MAC routing, `sndcreds`, and (on fleet) optional IP allowlist.

**Decision hinge:** both problems need materials vendors do **not** put in the PBX3 package. The operator (or MSP) must obtain **vendor-supplied root / client CAs** via partner portals, NDAs, or regional packs.

**Without those certs, local provisioning will always have issues** — phones that will not fetch over untrusted HTTPS, and/or a listener that cannot honestly claim “phones only.”

---

## Solo (singleton) — leave as shipped

Solo = one home, no fleet SBC provision edge. Phones hit the **instance** on `:41363` (FQDN or LAN IP per `provisionwith`).

**Locked:**

- No mandatory client mTLS on the home listener.
- Private-net GET-by-MAC remains acceptable for office / lab LAN.
- Prefer **`Once`** for passwords; after a handset factory reset, **reset provision state** on the extension so secrets send again.
- Prefer a **real server cert** (LE) when phones are picky; snakeoil is soak-only unless you push trust onto the phones.
- Do **not** expose solo `:41363` to the open internet without intent. Treat guest Wi‑Fi as hostile to the provision port.

**Why not force solo mTLS in product?** Solo is Rule-6 try-it / single-office LAN. A mandatory CA pack cannot ship cleanly; “optional mTLS” that still accepts bare curl helps nobody.

If remote desks must provision over the public internet from a singleton, that is already stretching solo — prefer fleet edge + CAs, or [M1 reseller/vendor full-config](phone-provisioning-m1.md) so PBX3 never serves the HTTP stream.

---

## Fleet edge — higher bar, still CA-gated

```text
Phone → https://provision.{apex}:41363/…   (HTTPS + optional mTLS + optional CIDR allowlist)
      → edge proxy_pass HTTP → home :41363   (SBC IPs only)
      → home expands stream from local SQLite
```

| Control | Role |
|---------|------|
| Stable `provision.{apex}` | Phones never learn home URLs (tenant move = map update, not RPS edit) |
| Vendor client-cert verify | Mitigates “anyone who knows a MAC” on the **public** edge |
| [Provision access](phone-provisioning-access.md) | Optional UFW allowlist on **`:41363` only** (site egress CIDRs) |
| Home firewall | Edge→home HTTP; provision port not a public desk target |

Even on fleet: **missing a brand’s CA** means that brand cannot join a **strict** mTLS posture. For those brands use Once + network controls, manual URL soak on a trusted net, or vendor-cloud M1 — not “mTLS optional that accepts everything.”

---

## Vendor CA reality (operator-facing)

Vendors often **claim** CAs are freely available. Practice is uneven: partner portals, NDAs, regional packs, silent rotations.

| Brand (indicative) | Rough reality |
|--------------------|---------------|
| **Yealink / Snom** | Partner / ops packs exist; lab mTLS has been proven when the CA is tipped on the edge |
| **Panasonic / Poly** | Obtainable with enough partner access; Poly often has clearer public PKI docs |
| **Grandstream** | Community / partner material varies — confirm before treating as production mTLS |
| **Fanvil** | No useful public Root CA story for edge verify; vendor docs push trust **onto** the phone; lean vendor discovery cloud for remote |
| **Gigaset** | Typically partner-only |

PBX3 will **not**:

- Bundle a complete third-party CA catalogue in the OSS package
- Ship a SPA “upload all OEM roots” panel as a v1 requirement
- Document pretend-mTLS for brands with no verifiable CA

PBX3 **will**:

- Keep optional edge mTLS for brands you can tip
- Keep `sndcreds`, audit masking, fail-closed MAC routing, and fleet allowlist as first-class controls
- Document that **you** source CAs for the brands you sell

---

## Secrets, audit, and preview

| Surface | Cleartext? | Notes |
|---------|------------|--------|
| **Wire to phone** | Yes when `Always` / unspent `Once` | Required for the handset to learn the password |
| **SPA expand preview** | Yes | Same privileged admin who can view/regen the SIP password |
| **Provision audit log** | Masked | Including OEM keys such as `SIP_PASS` — do not conflate with preview |

Prefer **`Once`** so daily re-polls do not re-emit passwords. Factory-reset phone + forgotten server-side reset → phone fetches config **without** password → REGISTER fails until you reset provision state.

Until desk SIP is widely TLS, passwords can still appear on **UDP signalling** — that does not excuse leaving cleartext in audit files.

---

## Practical harden checklist (no full mTLS)

1. Prefer **`Once`** (or `No` after enroll) on extensions.
2. After factory reset / handset swap: **Reset provision state**, then let the phone fetch once.
3. Put desks and `:41363` on a **voice / management VLAN**; block guest SSID to the provision port.
4. Solo: LE on the instance FQDN when phones reject snakeoil; avoid publishing `:41363` on WAN.
5. Fleet: tip vendor CAs you actually have → enable client verify; turn on [Provision access](phone-provisioning-access.md) when site egress CIDRs are stable.
6. Brands with no CA: M1 / vendor cloud, or LAN-only soak — do not advertise “mTLS protected.”
7. Treat browser GET-by-MAC as **admin debug on a trusted LAN**, not a public self-service URL.

---

## FAQ

**Why can I open the provision URL in a browser and see the config?**  
Because solo (and open edge) GET-by-MAC is a locator. Same class of privilege as an admin viewing the SIP password. Close the network path or add real client auth (mTLS + CA you own).

**Why doesn’t PBX3 just include the phone CAs?**  
Licensing, partner NDAs, regional packs, and silent vendor changes. Shipping a stale or incomplete pack is worse than documenting the gap.

**We only use vendor cloud config — do we care?**  
See [M1](phone-provisioning-m1.md). If phones never hit PBX3 `:41363`, this page’s HTTP risks do not apply; SIP Commit still matters.

**Is solo safe enough for a small office?**  
On a private LAN with `Once`, VLAN discipline, and no WAN exposure of `:41363`, yes as an assist. It is not a hardened internet-facing provision appliance — use fleet edge + CAs or M1 for that shape.

**Does mTLS stop MAC guessing across tenants?**  
It raises the bar to “needs a vendor client cert.” Cross-tenant safety still depends on MAC→tenant routing (fail closed) and never serving another cluster’s MAC.

---

## Summary

| Keep | Do not expect |
|------|----------------|
| Provision as LAN/fleet **assist** | Provision as the product security spine |
| Solo leave-as-is + network + `Once` | Forced solo home mTLS without CAs |
| Operator-sourced vendor roots/client CAs | A complete OEM CA pack inside PBX3 |
| Fleet optional mTLS + allowlist when tipped | “Optional mTLS” that still accepts bare curl |

Strong remote harden needs **vendor CAs the operator must supply**. Without them, local provisioning will keep showing rough edges — and that is an industry constraint, not a missing checkbox in the installer.
