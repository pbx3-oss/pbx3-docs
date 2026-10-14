# Install SBC edge (pbx3sbc + admin)

Greenfield install of the **SIP edge** (`pbx3sbc` / OpenSIPS + MariaDB) and the **Filament admin** (`pbx3sbc-admin`) on **Ubuntu 24.04 amd64**.

!!! note "What “SBC” means here"
    PBX3’s SBC is a **signaling border / proxy-registrar** (stable SIP front door: REGISTER, DID routing, Peers, WSS). **Media stays on the home** (RTP bypass) — not a B2BUA with transcoding. For classic media-terminating / Teams-certified SBC needs, peer a commercial box ahead (Rule 7).

This is **not** [SBC backup and restore](sbc-backup-restore.md) (cold DR) and **not** an HA promote drill — but the same install is the baseline for an active–passive pair (shared EIP / VIP as `advertised_address`).

!!! note "Architecture"
    OpenSIPS packages from `apt.opensips.org` on Ubuntu 24.04 are **amd64 (x86_64)** today. Use `t3.*` (or other x86_64), **not** `t4g`, unless you build OpenSIPS yourself.

## Prerequisites

| Need | Notes |
|------|--------|
| Host | Fresh **Ubuntu 24.04** x86_64, SSH as a sudoer (lab: `ubuntu`) |
| Disk | Prefer **≥ 16 GB** root; ~8 GB works but is tight after apt + PHP |
| Network | Outbound HTTPS for apt + GitHub clones |
| DNS (admin HTTPS) | A record for the public FQDN → **stable** public IPv4 (EIP) |
| Security group / firewall | **UDP/TCP 5060** (SIP), **TCP 22**, and for admin/LE: **TCP 80 + 443** from the internet (validators need world `80/443`) |

## Install order

1. Install **edge** (`pbx3sbc`) — OpenSIPS, MariaDB, firewall helpers  
2. Install **admin** (`pbx3sbc-admin`) — PHP/nginx/Filament (**`--server-name` required**)  
3. Smoke: services + admin login  
4. HTTPS: installer `--letsencrypt` **or** Filament **Certificates** (SPA-like)  
5. Optional: **TOTP 2FA** for Filament admins (authenticator app)  
6. Optional: seed peers/domains, or [restore a backup](sbc-backup-restore.md)

---

## 1. Clone and install the edge

```bash
sudo apt-get update && sudo apt-get install -y git curl
cd ~
git clone https://github.com/pbx3-oss/pbx3sbc.git
cd pbx3sbc
sudo ./install.sh --advertised-ip <PUBLIC_EIP_OR_VIP> --db-password '<DB_PASSWORD>'
```

When prompted **Reinitialize database? (y/N):** answer **`y`** on a fresh host.

Check: `systemctl is-active opensips mariadb` → both `active`.

---

## 2. Clone and install the admin

**`--server-name <PUBLIC_FQDN>` is required** (nginx + `APP_URL`). Do not use the EC2 internal hostname.

```bash
cd ~
git clone https://github.com/pbx3-oss/pbx3sbc-admin.git
cd pbx3sbc-admin

sudo ./install.sh \
  --server-name <PUBLIC_FQDN> \
  --db-host localhost \
  --db-name opensips \
  --db-user opensips \
  --db-password '<DB_PASSWORD>' \
  --opensips-mi-url http://127.0.0.1:8888/mi \
  --admin-name Admin \
  --admin-email '<ADMIN_EMAIL>' \
  --admin-password '<ADMIN_PASSWORD>' \
  --site-timezone America/New_York
```

`--site-timezone` (or `PBX3_SBC_SITE_TIMEZONE`) sets Filament Home/CDR day buckets (`SiteTimezone`). Default when omitted: host `/etc/timezone`. Interactive installs prompt (Enter = host default). Does **not** run `timedatectl`. Federated preference: OS **UTC**; override site TZ only when operators need a non-UTC day bucket.

Optional HTTPS in the same pass (DNS + SG 80/443 ready):

```bash
sudo ./install.sh \
  --server-name <PUBLIC_FQDN> \
  --letsencrypt --email '<LE_EMAIL>' \
  ... # same DB / admin flags
```

Installer also applies Fail2ban/Certificates **sudoers** when `pbx3sbc` is present next to this clone.

### Certificates panel (SPA kinship)

Filament **Certificates** mirrors the instance SPA:

| Section | Behaviour |
|---------|-----------|
| **Let's Encrypt** | Hostname from `APP_URL` (readonly); email + **Get certificate**; when configured: status + **Renew now** |
| **Purchased certificate** | Upload fullchain + key; Install / Remove |

Edge vs SPA: **no** multi-tenant SAN sync on the SBC admin host (single admin FQDN). Instance nodes: fleet = instance-only LE.

---

## 3. HTTPS after HTTP install

Filament → **Certificates** → **Get certificate**, or:

```bash
sudo ~/pbx3sbc-admin/scripts/le-admin-cert.sh setup \
  <PUBLIC_FQDN> <LE_EMAIL> /home/ubuntu/pbx3sbc-admin/public
```

Details: `pbx3sbc/workingdocs/LE_HTTPS_SBC_ADMIN.md`. SIP TLS out of scope.

---

## 3a. WebRTC / WSS (edge SIP-over-WSS) — not on by default

Desk phones use **UDP 5060**. **Browser WebRTC** needs **WSS on the SBC** (`wss://<edge-fqdn>:8089/ws`). The OpenSIPS template ships that block **commented out**.

After admin HTTPS / LE for the edge FQDN (same name is fine for WSS certs):

```bash
cd ~/pbx3sbc
sudo ./scripts/setup-opensips-wss.sh --cert-domain <PUBLIC_FQDN> --install-packages
# Then enable W1 in /etc/opensips/opensips.cfg: modules/socket + EXACTLY ONE cert pair
# (never both template examples — dual pairs → OpenSIPS will not start).
# Open SG + host firewall TCP 8089, opensips -C, restart opensips; confirm UDP 5060 still up.
```

Checklist: **`pbx3sbc/workingdocs/~/GiT/pbx3-ops/devdocs/oss-move/WEBRTC_W1_SBC_LAB.md`**. Lab LAN (self-signed, one command): [Install the Lab SBC](../installation/install-lab-sbc.md) §4 → `scripts/enable-lab-wss.sh`.

SPA **Line test** defaults to `wss://sbc.pbx3.com:8089/ws` — override to your edge FQDN when different.

---

## 4. Admin TOTP 2FA (optional, recommended)

Filament admin supports **opt-in** authenticator-app MFA (TOTP). Any compliant app works (2FAS, Authy, Google Authenticator, …). **No SMS.** Fleet Bearer API (`/api/fleet/*`) is separate and does **not** use this MFA.

### Enroll

1. Sign in with email + password (installer does **not** prompt for 2FA at create-admin).
2. Topbar **Profile** (next to Logout — the Filament avatar menu is hidden for SPA kinship).
3. Enable **Two Factor Authentication** → scan the QR → confirm with a 6-digit code.
4. **Save the recovery codes** shown once.
5. Next login: password → challenge → authenticator code (or a recovery code).

### Authenticator issuer label

The QR **issuer** defaults to **`Aelintra SBC`**. Override **before enroll** (or disable + re-enroll after changing):

```bash
# in ~/pbx3sbc-admin/.env
PBX3_TOTP_ISSUER="Aelintra SBC"
# HA pair example — distinct labels per member:
# PBX3_TOTP_ISSUER="Aelintra SBC active"
# PBX3_TOTP_ISSUER="Aelintra SBC standby"
```

Then `php artisan config:clear` (and reload PHP-FPM if needed).

Use a **distinct** issuer on each edge host (e.g. active vs standby) so the same admin email does not look identical in the authenticator app. Crypto still works with a shared label; UX does not.

There is **no** enroll-time UI to set the issuer — change `.env` first.

### Lockout

Prefer a recovery code. Sole admin locked out: clear 2FA for that user (tinker / `breezy_sessions`), then re-enroll. Detail: `pbx3sbc-admin/workingdocs/TOTP_2FA_SBC.md`.

Instance SPA Sanctum MFA is a separate track (not this edge admin).

---

## Active–passive HA pair (lab shape)

| Member | Owns EIP? | `advertised-ip` | `--server-name` | `--letsencrypt` |
|--------|-----------|-----------------|-----------------|-----------------|
| Active | Yes | **EIP** | Shared FQDN | **Yes** |
| Standby | No | **Same EIP** | Same FQDN | **No** (issue LE only after it owns the EIP) |

Promote = fence old active → reassociate EIP → **Let’s Encrypt on the new active** → confirm `https://<FQDN>/admin/login`. SIP alone is not enough.

After standby greenfield, run **warm-sync bootstrap** so Fleet **Sync now** works (fleet token, `log-ship.env`, AWS CLI, IAM `pbx3-sbc`, sudoers). Checklist: [SBC HA promote](sbc-ha-promote.md) Phase A; helper `pbx3sbc/scripts/bootstrap-ha-standby-warm.sh` + `check-ha-standby-ready.sh`.

Full gated checklist: **[SBC HA warm sync and EIP promote](sbc-ha-promote.md)**.

## Related

- [SBC HA warm sync and EIP promote](sbc-ha-promote.md)
- [SBC backup and restore](sbc-backup-restore.md)
- [Fleet overview](overview.md)
