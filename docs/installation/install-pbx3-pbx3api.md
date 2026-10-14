# Install pbx3 and pbx3api

!!! tip "AI-assisted path"
    Prefer an agent to drive this page: [AI-assisted install](ai-assisted-install.md) (solo kickoff). You still approve DNS / VM / LE gates.

Bring up a **clean Ubuntu 24.04** node (EC2, VM, or bare metal) with public DNS and TLS: packages → identity → API → DNS → Let’s Encrypt → prove `/up` → SPA admin.

This is the **same stack** as the [Lab home](install-lab-home.md) install (`install-home-host.sh`), plus public DNS and LE. It does **not** run fleet onboard or Provision edge.

| After this page | Continue at |
|-----------------|-------------|
| Stay solo | [Solo trial](../getting-started/solo-trial.md) |
| **New fleet home** (you just finished Install) | [Commission a fleet instance](../fleet/commission-instance.md) → **Step 1 Adopt** |
| Node already up; adopt only | [Onboard](../fleet/onboard-instance.md) |
| LAN lab (no DNS / LE) | [Install the Lab home PBX](install-lab-home.md) |

!!! tip "New EC2 into the fleet end-to-end"
    Do this Install page first (includes bringing up the host for LE). Then [Commission](../fleet/commission-instance.md) for onboard + edge — Commission does not repeat EC2/DNS/LE.

!!! note "Fleet timing (cloud vs lab)"
    **Skip the fleet service token** on this page (press Enter when prompted, or omit `PBX3_FLEET_SERVICE_TOKEN`). [Commission](../fleet/commission-instance.md) / onboard writes fleet `.env` and Egress. Lab home seeds fleet at install time because there is no separate Mac onboard step.

## What you are installing

| Piece | Form | Lands on the node as |
|-------|------|----------------------|
| **pbx3** | `.deb` (via `install-home-host.sh`) | `/opt/pbx3` — Asterisk configs, SQLite, scripts |
| **pbx3api** | **git tree** (no `.deb`) | `/opt/pbx3api` — nginx + PHP-FPM API on **:44300** |
| **pbx3cagi** | `.deb` (amd64; still private) | CAGI binary — install after the home script (see [CAGI](#cagi-needed-for-a-call)) |

There is **no** monorepo. GitHub has separate repos. Release `.deb`s live at the **root of `pbx3` and `pbx3cagi` on `main`**. **pbx3api** is public; **pbx3** / **pbx3cagi** are still private today.

**Package floor (adjust when newer):** prefer the newest `pbx3_*.deb` on `main` (script picks the highest version next to the script). Cloud lab floor reference: `pbx3_0.0.5-5` / rebuild `0.0.5-6` · `pbx3cagi_1.0.0-18`.

---

## Before you start

| Have it? | Thing |
|:--------:|-------|
| ☐ | Ubuntu **24.04** node with sudo (bare metal, VM, or EC2) |
| ☐ | SSH access (`ubuntu@…` or equivalent) + private key if needed |
| ☐ | Laptop with **git**, **scp**/**rsync**, and network access to clone **`pbx3-oss/pbx3`** (and **`pbx3cagi`** for calls) — public HTTPS, no GitHub login required |
| ☐ | Inbound **22**, **80** (LE), **44300**; outbound **443** — see [Requirements](requirements.md) |
| ☐ | DNS control for an apex (e.g. `pbx3.com`) **or** readiness to create an **A** record after install |
| ☐ | Email for Let’s Encrypt + friendly site **Name** + admin SPA email/password you invent (**8+** characters; not a docs placeholder) |

### Worksheet

Use **straight** quotes only. Put comments on their **own** lines — zsh often treats `# …` on the same line as `export` as more arguments.

```bash
# SSH key path (omit KEY_FILE later if your agent already has the key)
export KEY_FILE=/path/to/your.pem

# SSH target
export SSH_HOST=ubuntu@YOUR.PUBLIC.IP

export LE_EMAIL=you@example.com

# Friendly Name — not the hostname
export SITE_NAME='My node'

# Apex; installer mints opaque shortuid → fqdn={shortuid}.{apex}
export DOMAIN_TLD=pbx3.com

# SPA admin (required for non-interactive install)
export ADMIN_EMAIL=you@example.com
export ADMIN_PASSWORD='choose-a-strong-password'

# After install (Step 3), from sqlite:
# export INSTANCE_FQDN=…
# export KSUID=…
# export SHORTUID=…
```

---

## Step 1 — Get sources on the node

The home installer expects a **`pbx3` tree** that contains `scripts/install-home-host.sh`, a `pbx3_*.deb` on `main`, and **`pbx3api`** nested as `pbx3/pbx3api` (or already at `/opt/pbx3api`).

### Prefer: clone on the node when repos are reachable

**pbx3api** is public. If **pbx3** clone works on the node:

```bash
ssh -i "$KEY_FILE" "$SSH_HOST"
# on the node:
sudo apt-get update && sudo apt-get install -y git
cd ~
git clone --depth 1 https://github.com/pbx3-oss/pbx3.git
git clone --depth 1 https://github.com/pbx3-oss/pbx3api.git pbx3/pbx3api
```

### Usual lab path while pbx3 is private

Copy the tree (including the `.deb` on `main`) from a laptop that already has GitHub auth:

```bash
# laptop — refresh tips
cd ~/GiT/pbx3-master/pbx3 && git checkout main && git pull --ff-only
cd ~/GiT/pbx3-master/pbx3api && git checkout main && git pull --ff-only

# copy pbx3 (deb + install-home-host.sh) then nest pbx3api
rsync -az --exclude .git -e "ssh -i $KEY_FILE" \
  ~/GiT/pbx3-master/pbx3/ "${SSH_HOST}:~/pbx3/"
rsync -az --exclude .git -e "ssh -i $KEY_FILE" \
  ~/GiT/pbx3-master/pbx3api/ "${SSH_HOST}:~/pbx3/pbx3api/"
```

Omit `-e "ssh -i $KEY_FILE"` if your agent / default key already works.

---

## Step 2 — Run the home installer (skip fleet)

On the **node**:

```bash
cd ~/pbx3
sudo DOMAIN_TLD="${DOMAIN_TLD}" \
  INSTANCE_SITENAME="${SITE_NAME}" \
  PBX3_ADMIN_EMAIL="${ADMIN_EMAIL}" \
  PBX3_ADMIN_PASSWORD="${ADMIN_PASSWORD}" \
  ./scripts/install-home-host.sh
```

Or interactive: `sudo ./scripts/install-home-host.sh` — answer apex (Enter → `pbx3.com`), site name, admin email/password, then **Enter to skip fleet** when asked for the fleet service token.

The script:

- installs the newest `pbx3_*.deb` it finds next to the script (or set `PBX3_DEB=…`)
- runs `/opt/pbx3/scripts/installer.sh` (mints `{shortuid}.{apex}`; no vanity FQDN)
- installs **pbx3api** and runs its installer (`migrate --force` as **www-data**; writable Laravel log)
- leaves API on **snakeoil** `:44300` (LE is Step 5)
- applies **UFW** baseline (**fleet:** SSH :22 + API :44300 from **`any`** — narrow later; **solo:** those plus SIP/RTP from **LAN CIDR** — see [Firewall](../admin/firewall.md))

!!! warning
    Do **not** pass a fleet token here if this box will [Commission](../fleet/commission-instance.md) next — onboard owns `PBX3_ORG_BUCKET` / token / Egress.  
    Do **not** run `reloader.sh` after a clean first install.  
    Do **not** pass vanity `INSTANCE_FQDN=kildare.pbx3.com` unless you intentionally set `PBX3_ALLOW_VANITY_FQDN=1`.

**Clean re-install** (same host, wipe DB identity):

```bash
cd ~/pbx3
sudo PBX3_CLEAN_INSTALL=1 \
  DOMAIN_TLD="${DOMAIN_TLD}" INSTANCE_SITENAME="${SITE_NAME}" \
  PBX3_ADMIN_EMAIL="${ADMIN_EMAIL}" PBX3_ADMIN_PASSWORD="${ADMIN_PASSWORD}" \
  ./scripts/install-home-host.sh
```

---

## Step 3 — Record identity

The installer mints these; they are **not** the SSH nickname. Capture before DNS / LE:

```bash
sqlite3 /opt/pbx3/db/sqlite.db \
  "SELECT id, shortuid, fqdn, sitename FROM globals WHERE pkey='global';"

sqlite3 /opt/pbx3/db/sqlite.db "SELECT id,email FROM users;"
# must show at least one row — your email, not a docs placeholder

export KSUID=$(sqlite3 /opt/pbx3/db/sqlite.db "SELECT id FROM globals WHERE pkey='global';")
export SHORTUID=$(sqlite3 /opt/pbx3/db/sqlite.db "SELECT shortuid FROM globals WHERE pkey='global';")
export INSTANCE_FQDN=$(sqlite3 /opt/pbx3/db/sqlite.db "SELECT fqdn FROM globals WHERE pkey='global';")
echo "KSUID=$KSUID"
echo "SHORTUID=$SHORTUID"
echo "INSTANCE_FQDN=$INSTANCE_FQDN"
```

Do **not** continue until `echo "$INSTANCE_FQDN"` prints a real name (e.g. `xhcjkh.pbx3.com`). Empty → `dig` fails with `'' is not a legal name`.

If `users` is empty (older floor debs), create an admin:

```bash
sudo PBX3_ADMIN_EMAIL="${ADMIN_EMAIL}" \
  PBX3_ADMIN_PASSWORD="${ADMIN_PASSWORD}" \
  /opt/pbx3/scripts/bootstrap-admin-user.sh
```

---

## Step 4 — Prove the stack (before DNS)

**On the node**, use loopback — not the public FQDN:

```bash
curl -k -sS -o /dev/null -w "%{http_code}\n" https://127.0.0.1:44300/up
# expect 200

sudo systemctl is-active nginx php8.3-fpm asterisk
```

**Stop if not 200.** DNS and LE will not fix a broken API.

!!! warning "AWS hairpin — do not curl the public FQDN from the node"
    On EC2, `curl https://$INSTANCE_FQDN:44300/up` **from the instance itself** usually **times out**. The box cannot open TCP to its own public IP/EIP. That is normal.

    | Where | Command |
    |-------|---------|
    | **On the node** | `https://127.0.0.1:44300/up` (use `-k` until LE) |
    | **Ops laptop / Mac** | `https://${INSTANCE_FQDN}:44300/up` |

    Also open **AWS security group TCP 44300** to your **ops public IP `/32`** (and Gatekeeper if fleet). Port **80** `0.0.0.0/0` only covers Let’s Encrypt — it does **not** open the API.

If nginx complains about a duplicate `default_server` on port 80:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

---

## Step 5 — DNS

Requires **`INSTANCE_FQDN`** from [Step 3](#step-3--record-identity). If unset, stop and export it — do not dig an empty name.

| Name | Type | Value |
|------|------|--------|
| Exact `globals.fqdn` (e.g. `xhcjkh.pbx3.com`) | **A** | Node public IP / EIP |

```bash
test -n "$INSTANCE_FQDN" || { echo "INSTANCE_FQDN not set — re-run Step 3 exports"; exit 1; }
echo "Checking DNS for $INSTANCE_FQDN"
dig +short "$INSTANCE_FQDN"
# must equal this node’s public IP
```

Fleet nodes do **not** publish public tenant A records. Solo multi-tenant Option A is a different TLS story — see [TLS overview](../tls/overview.md).

---

## Step 6 — First Let’s Encrypt certificate

Port **80** must reach this host from the internet. DNS A must already match.

```bash
sudo /opt/pbx3/scripts/le-instance-bootstrap.sh "$LE_EMAIL"
# staging: sudo PBX3_LE_STAGING=1 /opt/pbx3/scripts/le-instance-bootstrap.sh "$LE_EMAIL"
```

Or SPA → **Certificates → Get certificate** after snakeoil login. Full notes: [First Let's Encrypt certificate](../tls/first-letsencrypt.md).

**Prove trusted HTTPS from the ops laptop** (not from the node — see hairpin warning in Step 4). SG must allow your Mac’s public IP on **44300**:

```bash
# On the Mac / ops workstation
curl -4 -sS --connect-timeout 5 --max-time 10 -o /dev/null -w "%{http_code}\n" "https://${INSTANCE_FQDN}:44300/up"
# expect 200 without -k

# On the node only:
curl -k -sS -o /dev/null -w "%{http_code}\n" https://127.0.0.1:44300/up
```

Open the shared Admin SPA at **[https://app.pbx3.com](https://app.pbx3.com)** (you do **not** install or build **pbx3spa** on this node) and [sign in](../getting-started/sign-in.md) with the email/password from the worksheet. API base URL example: `https://${INSTANCE_FQDN}:44300/api`. Node API must allow CORS from `https://app.pbx3.com` (default lab installs usually allow `*`).

---

## CAGI (needed for a call)

**pbx3cagi** is still private. On **amd64** cloud nodes, copy the release `.deb` from a laptop that has the repo and install it:

```bash
# laptop
cd ~/GiT/pbx3-master/pbx3cagi && git checkout main && git pull --ff-only
# use the newest pbx3cagi_*.deb on main (floor reference: 1.0.0-18)
scp -i "$KEY_FILE" pbx3cagi_1.0.0-18_all.deb "${SSH_HOST}:/tmp/"

# node
sudo apt install -y /tmp/pbx3cagi_1.0.0-18_all.deb
dpkg-query -W pbx3cagi
```

Lab ARM guests compile from source — see [Lab home → CAGI](install-lab-home.md#cagi-needed-for-a-call).

---

## Checklist

- [ ] `pbx3` tree + `pbx3api` on the node; `install-home-host.sh` succeeded (fleet skipped)
- [ ] KSUID / opaque FQDN / admin user recorded
- [ ] `GET https://127.0.0.1:44300/up` → **200** (on node)
- [ ] DNS **A** for `globals.fqdn` → public IP
- [ ] Let’s Encrypt applied
- [ ] From **ops laptop**: trusted `https://{fqdn}:44300/up` → **200** (SG allows your IP on 44300)
- [ ] Admin SPA login works from **https://app.pbx3.com** (laptop)
- [ ] (Calls) `pbx3cagi` installed on amd64

## Failure cheat sheet

| Symptom | Likely fix |
|---------|------------|
| On node, `curl https://$INSTANCE_FQDN:44300/up` hangs | Expected hairpin — use `127.0.0.1` on the node; public URL from the laptop |
| Laptop curl to `:44300` times out (~5–130s) | SG: add your public IP `/32` on **44300** (port 80 open is not enough) |
| LE fails `example.com` contact | Use a real email — Let’s Encrypt rejects `*.example.com` |
| `/up` not 200 on localhost | home installer / nginx default site / PHP-FPM — re-check Step 2–4 |
| SPA login fails / `users` empty | Run `bootstrap-admin-user.sh` with **your** email (see Step 3) |
| SPA “Cannot reach API” but `curl …/up` is 200 | Often login **500**: root-owned `storage/logs/laravel.log` or missing migrate columns. Fix: `sudo chown www-data:www-data /opt/pbx3api/storage/logs/laravel.log` and `sudo -u www-data php artisan migrate --force` |
| `sudo php artisan …` after install | Prefer `sudo -u www-data php artisan …` — root recreates the log-ownership trap |
| Fleet `.env` / Egress missing on a **fleet** home | Expected after this page — continue at [Commission](../fleet/commission-instance.md); do not paste the lab fleet-token install block here |

## Next

**Solo install complete.** If this box should join the fleet:

→ **[Commission a fleet instance — Step 1 Adopt](../fleet/commission-instance.md#step-1--adopt-into-fleet-onboard)**  
(`onboard-fleet-instance.sh` → Provision edge → Fail2ban)

| Goal | Go to |
|------|--------|
| Stay solo | [Solo trial](../getting-started/solo-trial.md) · [Admin guide](../admin/home-dashboard.md) |
| New fleet home (after Install) | [Commission](../fleet/commission-instance.md) |
| Adopt only | [Onboard](../fleet/onboard-instance.md) or Commission Step 1 |
| Same KSUID after EC2 loss | [Rebuild from S3](../fleet/rebuild-from-s3.md) |
| LAN lab (no DNS/LE) | [Lab home](install-lab-home.md) |

## Related

- [Requirements](requirements.md) · [First-run installer](first-run.md) · [Upgrade a package](upgrade.md)
- Lab: [operator worksheet](install-lab-worksheet.md) · [Lab home](install-lab-home.md)
