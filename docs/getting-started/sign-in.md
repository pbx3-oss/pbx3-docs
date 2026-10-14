# Sign in to the admin UI

**Open the SPA at [https://app.pbx3.com](https://app.pbx3.com)** unless you are on the [Lab Vite path](../installation/install-lab-spa.md) (`http://localhost:5173`).

PBX3 uses **one SPA** with two auth planes:

| Mode | Token | Used for |
|------|-------|----------|
| **Instance admin** | Sanctum (per node) | Tenants, extensions, Commit, Backup, Certificates |
| **Fleet console** | Gatekeeper session | Instances, Jobs, DIDs, Reconcile, Users |

Keep those separate (do not mix tokens).

## Instance admin (day-to-day)

### Solo

1. Open **https://app.pbx3.com**.
2. Email + password + API URL (`https://{node}:44300/api`).
3. No catalog required.

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
