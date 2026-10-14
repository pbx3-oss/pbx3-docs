# Cannot log in

| Symptom | Check |
|---------|-------|
| Empty catalog / load fail | Public catalog URL; key `catalog/instance-index.json`; bucket CORS allows **`https://app.pbx3.com`**; BPA |
| Wrong fleet / stale catalog | Login **Switch fleet catalog…** or `?catalog=`; reset to default if needed |
| API “Load failed” | SG **44300**; UFW; `curl -k https://NODE:44300/up` → 200; API CORS allows SPA origin |
| Browser TLS warning | Node still snakeoil / wrong SAN — [First LE](../tls/first-letsencrypt.md); Lab snakeoil → use [Vite](../installation/install-lab-spa.md) not Pages |
| Solo confusion | No catalog; API URL only on **app.pbx3.com** |
| Fleet gate stuck | Gatekeeper up (`/health`); fleet user enabled; Exit then Sign in again |
| Break-glass only works | Prefer email/password; rotate/revoke pasted tokens after ops |

Shared SPA: `https://app.pbx3.com` · Lab gatekeeper: `https://control.pbx3.com` · golden API: `https://08jzwn.pbx3.com:44300/api`.
