# Solo trial (one node)

Run **one** PBX3 instance without fleet catalog, org S3, Gatekeeper, or SBC.

**Install the stack first:** [Install pbx3 and pbx3api](../installation/install-pbx3-pbx3api.md) (`install-home-host.sh` → DNS → LE). This page is sign-in and solo scope only.

## What you need

| Need | Do not need |
|------|-------------|
| Ubuntu 24.04 node with `pbx3` + `pbx3api` | Org S3 bucket / instance catalog |
| Shared admin SPA (**[app.pbx3.com](https://app.pbx3.com)**) | Your own SPA host or Pages deploy |
| Reachable API at `https://{fqdn}:44300/api` | Control host / Gatekeeper |
| Sanctum admin login | SBC (optional for internet SIP later) |

## Sign in

1. Open **[https://app.pbx3.com](https://app.pbx3.com)** (Lab LAN snakeoil: use [Vite](../installation/install-lab-spa.md) instead).
2. Enter **email** and **password** (instance admin).
3. Enter **API base URL**, e.g. `https://YOURNODE.pbx3.com:44300/api`.
4. Do **not** point at a fleet catalog (picker unused / solo path).

## Lab tip

Golden example API:

`https://08jzwn.pbx3.com:44300/api`

Verify TLS without the SPA proxy:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' https://08jzwn.pbx3.com:44300/up
# expect 200 (no -k)
```

When you outgrow solo, see [Fleet overview](../fleet/overview.md) and [Commission a fleet instance](../fleet/commission-instance.md) / [Onboard](../fleet/onboard-instance.md).
