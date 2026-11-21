# Solo trial (one node)

Run **one** PBX3 instance without fleet catalog, org S3, Gatekeeper, or SBC.

**Install first:** follow the linear **[Solo install](../installation/install-solo.md)** (Packagecloud `pbx3` → first-run → GitHub `pbx3api` → DNS/LE → SPA).

## What you need

| Need | Do not need |
|------|-------------|
| Ubuntu 24.04 node with `pbx3` + `pbx3api` | Org S3 bucket / instance catalog |
| Shared admin SPA (**[app.pbx3.com](https://app.pbx3.com)**) | Your own SPA host or Pages deploy |
| Reachable API at `https://{fqdn}:44300/api` | Control host / Gatekeeper |
| Sanctum admin login | SBC (optional for internet SIP later) |

## Sign in

After [Solo install](../installation/install-solo.md), open **[app.pbx3.com](https://app.pbx3.com)** → **Manage instance** → **Enter API URL manually** → `https://YOURNODE:44300/api` → admin email/password.

Full chooser detail (including LAN IP + snakeoil): **[Sign in → Solo](sign-in.md#solo-or-any-node-not-in-the-catalog)**.

Desk provisioning on solo stays LAN-assist (no forced mTLS) — [Provisioning security](../admin/phone-provisioning-security.md).

When you outgrow solo, see [Fleet overview](../fleet/overview.md) and [Commission a fleet instance](../fleet/commission-instance.md).
