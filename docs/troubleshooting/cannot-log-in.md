# Cannot log in

| Symptom | Check |
|---------|-------|
| Empty catalog / load fail | Public catalog URL; key `catalog/instance-index.json`; bucket CORS allows **`https://app.pbx3.com`**; BPA |
| Wrong fleet / stale catalog | Login **Switch fleet catalog…** or `?catalog=`; reset to default if needed |
| No place to type API URL | Chooser → **Manage instance** → **Enter API URL manually** — [Sign in (solo)](../getting-started/sign-in.md#solo-or-any-node-not-in-the-catalog) |
| “Cannot reach the PBX API” but `curl -k …/up` is 200 | **Snakeoil / untrusted cert:** open `https://NODE:44300/up` in the **same browser**, accept the warning, retry SPA login. Cloud-oriented hints (SG / SSH tunnel) in that error are often irrelevant on LAN. |
| API “Load failed” / timeout | SG **44300** (cloud); UFW allows your client; `curl -k https://NODE:44300/up` → 200; API CORS allows SPA origin (`*` is fine on lab) |
| Browser TLS warning on `/up` | Expected until [First LE](../tls/first-letsencrypt.md) or a custom cert; accept once for lab, or use [Vite](../installation/install-lab-spa.md) while developing |
| Solo confusion | No catalog required; manual API URL on **app.pbx3.com** as above |
| Fleet gate stuck | Gatekeeper up (`/health`); fleet user enabled; Exit then Sign in again |
| Break-glass only works | Prefer email/password; rotate/revoke pasted tokens after ops |

Shared SPA: `https://app.pbx3.com` · Lab gatekeeper: `https://control.pbx3.com` · golden API: `https://08jzwn.pbx3.com:44300/api`.
