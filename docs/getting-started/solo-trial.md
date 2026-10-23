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

Full chooser steps (including **LAN IP + snakeoil**): **[Sign in → Solo](sign-in.md#solo-or-any-node-not-in-the-catalog)**.

Short path:

1. Open **[https://app.pbx3.com](https://app.pbx3.com)**.
2. **Manage instance** → **Enter API URL manually**.
3. **API base URL** e.g. `https://YOURNODE.pbx3.com:44300/api` or `https://192.168.x.x:44300/api`.
4. Instance admin **email** + **password**.

If the SPA says it cannot reach the API but `curl -k https://HOST:44300/up` works from your laptop, open that `/up` URL in the browser once and accept the certificate warning, then retry. Details on the [Sign in](sign-in.md#first-visit-with-a-lab-snakeoil-certificate) page.

## Lab tip

Golden (cloud, trusted LE) example API:

`https://08jzwn.pbx3.com:44300/api`

```bash
curl -sS -o /dev/null -w '%{http_code}\n' https://08jzwn.pbx3.com:44300/up
# expect 200 (no -k) when LE is live
```

LAN lab before LE:

```bash
curl -k -sS -o /dev/null -w '%{http_code}\n' https://192.168.1.148:44300/up
# expect 200; then trust the same URL in the browser for app.pbx3.com
```

When you outgrow solo, see [Fleet overview](../fleet/overview.md) and [Commission a fleet instance](../fleet/commission-instance.md) / [Onboard](../fleet/onboard-instance.md).
