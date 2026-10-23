# Sign in to the admin UI

**Open the SPA at [https://app.pbx3.com](https://app.pbx3.com)** unless you are on the [Lab Vite path](../installation/install-lab-spa.md) (`http://localhost:5173`).

PBX3 uses **one SPA** with two auth planes:

| Mode | Token | Used for |
|------|-------|----------|
| **Instance admin** | Sanctum (per node) | Tenants, extensions, Commit, Backup, Certificates |
| **Fleet console** | Gatekeeper session | Instances, Jobs, DIDs, Reconcile, Users |

Keep those separate (do not mix tokens).

## Instance admin (day-to-day)

### Solo (or any node not in the catalog)

Use this for a **LAN / localhost** home, a fresh solo install, or any node that is not listed in the fleet catalog.

1. Open **[https://app.pbx3.com](https://app.pbx3.com)**.
2. On the chooser, click **Manage instance** (not Fleet console).
3. On the catalog picker, click **Enter API URL manually** (bottom of the list).
4. **API base URL:** `https://{host-or-ip}:44300/api`  
   Examples: `https://192.168.1.148:44300/api`, `https://5ks5v4.pbx3.com:44300/api`.
5. Email + password for that node’s Sanctum admin.

No catalog, Gatekeeper, or SBC is required.

#### First visit with a lab snakeoil certificate

Until the node has a **trusted** Let’s Encrypt (or custom) cert, the browser blocks `fetch` from **app.pbx3.com** to the node even when `curl -k` works. Trust the cert **once in the same browser**:

1. Open `https://{host-or-ip}:44300/up` in a new tab (same browser profile as the SPA).
2. Accept the certificate warning (Advanced → proceed / continue to site).
3. Confirm the page returns OK / HTTP success.
4. Return to **app.pbx3.com** and sign in again with the same API base URL.

That exception is per browser (and usually per hostname/IP). After [First Let's Encrypt](../tls/first-letsencrypt.md) on a public FQDN, this step is unnecessary.

**Lab alternative:** [Vite on your PC](../installation/install-lab-spa.md) proxies `/api` and ignores upstream TLS — useful while developing the SPA, not required for day-to-day solo admin.

### Fleet catalog picker

1. Open **https://app.pbx3.com** (default catalog is the lab/reference fleet when baked).
2. Other fleets: **Switch fleet catalog…** (or `?catalog=https://…/catalog/instance-index.json`). That bucket must allow CORS from `https://app.pbx3.com` — [SPA catalog CORS](../cloud/spa-catalog-cors.md).
3. **Refresh catalog** if needed → pick an instance.
4. Sign in with **that node's** admin credentials.
5. Top bar shows connected instance label + FQDN.

You do **not** build or host a SPA per fleet.

### Lab endpoints (reference)

| Role | URL |
|------|-----|
| Shared SPA | `https://app.pbx3.com` |
| Golden API | `https://08jzwn.pbx3.com:44300/api` |
| Second node | `https://bzy54n.pbx3.com:44300/api` |
| Catalog JSON | `https://08jzwn-pbx3.s3.us-east-1.amazonaws.com/catalog/instance-index.json` |
| Gatekeeper | `https://control.pbx3.com` |
| SBC admin | `https://sbc.pbx3.com/admin` |
| Lab Vite (LAN only) | `http://localhost:5173` |

## Fleet console

1. From SPA login chooser: **Fleet console**, or **Enter Fleet** (dual-hat) after instance login.
2. Sign in to Gatekeeper (email/password). Lab user: `fleet@pbx3.com` (password in ops secret store).
3. Nav stays locked until Sign in succeeds.
4. **Exit** returns to instance mode; **Logout** ends the fleet session.

Break-glass Bearer paste is ops-only under Advanced — prefer normal fleet login.

## Checks when login fails

See [Cannot log in](../troubleshooting/cannot-log-in.md).
