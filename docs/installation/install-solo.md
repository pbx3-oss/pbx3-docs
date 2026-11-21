# Solo install

Install **one** PBX3 node on a clean Ubuntu 24.04 box. No fleet catalog, org S3, Gatekeeper, or SBC.

When you finish this page you will have:

- packages on the host (`pbx3` + the admin **API**)
- a minted instance FQDN and Sanctum admin login
- HTTPS on the API port (`:44300`)
- the shared SPA at **[app.pbx3.com](https://app.pbx3.com)** talking to your node

That is the whole solo story. Fleet comes later, if you need it.

!!! note "Packagecloud"
    Debs ship from Packagecloud repo **`aelintra/pbx3`** (same pattern as SARK 6.5). The repo is the intended public path; create/publish it before treating the `curl | bash` step as live.

## Before you begin

PBX3 is meant to run as an **appliance**. Start from an Ubuntu **24.04** minimal install with little more than SSH. Do not drop it onto a busy multi-purpose server.

You need:

- sudo on the box
- inbound **22** (SSH), **80** (Let’s Encrypt), **44300** (API)
- outbound **443** (apt, Let’s Encrypt)
- DNS control for an apex (e.g. `pbx3.com`), **or** willingness to use a LAN IP with a snakeoil cert for a first look
- an email for Let’s Encrypt, a friendly site name, and an admin email/password you invent (**8+** characters)

You do **not** need org S3, a control host, an SBC, or your own SPA build.

## Prepare the box

```bash
sudo apt update
sudo apt upgrade
sudo reboot
```

After reboot, add the Packagecloud repo:

```bash
curl -s https://packagecloud.io/install/repositories/aelintra/pbx3/script.deb.sh | sudo bash
```

## Install pbx3

```bash
sudo apt install pbx3
```

This lands the backend under `/opt/pbx3` (Asterisk configs, SQLite, scripts, firewall helpers) and pulls **pbx3cagi** as a dependency (needed for calls). It does **not** yet give you a browser UI — the SPA talks to a separate API process.

## First-run identity

Mint the instance FQDN and create the SPA admin. On the node:

```bash
sudo DOMAIN_TLD=pbx3.com \
  INSTANCE_SITENAME='My PBX' \
  PBX3_ADMIN_EMAIL=you@example.com \
  PBX3_ADMIN_PASSWORD='choose-a-strong-password' \
  /opt/pbx3/scripts/installer.sh
```

Or run interactively (`sudo /opt/pbx3/scripts/installer.sh`) and answer the prompts. Use your real apex, site name, and admin credentials. Skip any fleet / service-token prompt (press Enter / leave empty) — this is solo.

The installer mints an opaque hostname and FQDN `{shortuid}.{apex}` (for example `5ks5v4.pbx3.com`). Record them:

```bash
sqlite3 /opt/pbx3/db/sqlite.db \
  "SELECT id, shortuid, fqdn, sitename FROM globals WHERE pkey='global';"

sqlite3 /opt/pbx3/db/sqlite.db "SELECT id,email FROM users;"
```

Export for later steps:

```bash
export INSTANCE_FQDN=$(sqlite3 /opt/pbx3/db/sqlite.db "SELECT fqdn FROM globals WHERE pkey='global';")
echo "$INSTANCE_FQDN"
```

Do not continue until that prints a real name. Empty FQDN breaks DNS and Let’s Encrypt.

If `users` is empty (unusual on a current floor), create the admin:

```bash
sudo PBX3_ADMIN_EMAIL=you@example.com \
  PBX3_ADMIN_PASSWORD='choose-a-strong-password' \
  /opt/pbx3/scripts/bootstrap-admin-user.sh
```

## Install the API (from GitHub)

The admin SPA at **app.pbx3.com** is a static site. Every Save, Commit, and login hits **pbx3api** on your node at `https://…:44300`. That API is not a Packagecloud `.deb` yet, so install it from GitHub.

On the node (after `/opt/pbx3/db/sqlite.db` exists):

```bash
sudo apt-get install -y git
sudo git clone --depth 1 https://github.com/pbx3-oss/pbx3api.git /opt/pbx3api
sudo rm -f /etc/nginx/sites-enabled/default
sudo /opt/pbx3api/scripts/installer.sh
```

The API installer pulls nginx + PHP-FPM, puts snakeoil TLS on **:44300**, and links Laravel to `/opt/pbx3/db/sqlite.db`.

## Prove the stack

On the **node**, use loopback (not the public FQDN — EC2 often cannot hairpin to its own public IP):

```bash
curl -k -sS -o /dev/null -w "%{http_code}\n" https://127.0.0.1:44300/up
# expect 200

sudo systemctl is-active nginx php8.3-fpm asterisk
```

**Stop if `/up` is not 200.** DNS and certificates will not fix a broken API.

LAN-only trial: you can sign in now with `https://YOUR.LAN.IP:44300/api` and a snakeoil trust step in the browser (see [Sign in](../getting-started/sign-in.md#first-visit-with-a-lab-snakeoil-certificate)). For a public solo box, continue with DNS and Let’s Encrypt.

## DNS

Create an **A** record for the exact minted FQDN → this node’s public IP.

```bash
test -n "$INSTANCE_FQDN" || { echo "INSTANCE_FQDN not set — re-run the sqlite export above"; exit 1; }
dig +short "$INSTANCE_FQDN"
# must equal this node’s public IP
```

## Let’s Encrypt

Port **80** must reach the host from the internet. DNS must already match.

```bash
sudo /opt/pbx3/scripts/le-instance-bootstrap.sh you@example.com
```

From your **laptop** (not from the node):

```bash
curl -4 -sS --connect-timeout 5 --max-time 10 -o /dev/null -w "%{http_code}\n" \
  "https://${INSTANCE_FQDN}:44300/up"
# expect 200 without -k
```

If that times out on AWS, open security-group **TCP 44300** to your public IP `/32`. Opening port 80 for Let’s Encrypt does not open the API.

## Open the SPA

You do **not** install a UI on the node. Use the shared admin app:

1. Open **[https://app.pbx3.com](https://app.pbx3.com)**.
2. **Manage instance** → **Enter API URL manually**.
3. API base URL: `https://YOUR_FQDN:44300/api` (or `https://YOUR.LAN.IP:44300/api` before LE).
4. Sign in with the admin email and password from first-run.

If the SPA cannot reach the API but `curl -k …/up` works, open `https://HOST:44300/up` in the same browser once and accept the certificate warning, then retry.

Desk phones and trunks are admin work after login — start from the [Home dashboard](../admin/home-dashboard.md).

## Checklist

- [ ] Packagecloud repo added; `pbx3` installed (brings in `pbx3cagi`)
- [ ] First-run identity + admin user recorded
- [ ] `pbx3api` cloned to `/opt/pbx3api` and installer succeeded
- [ ] `GET https://127.0.0.1:44300/up` → **200** on the node
- [ ] DNS **A** for `globals.fqdn` → public IP (if public)
- [ ] Let’s Encrypt applied; laptop trusted `/up` → **200**
- [ ] SPA login from **app.pbx3.com**

## Next

**Solo install complete.** Stay on this box and configure tenants, extensions, and trunks in the SPA.

When you outgrow a single node:

- [Fleet overview](../fleet/overview.md)
- [Commission a fleet instance](../fleet/commission-instance.md)

For a longer git/`install-home-host.sh` path (lab copies, fleet-prep overlap), see [Install pbx3 and pbx3api](install-pbx3-pbx3api.md).
