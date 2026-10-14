# SBC backup and restore

Cold DR for the **SBC edge** (OpenSIPS + Filament admin). Catalog re-project recovers fleet-owned routing only — **not** carrier peers, Fail2ban lists, or admin users. Those live in the edge MariaDB dump.

This is **not** [instance backups](backups-org-bucket.md) and **not** [SBC MySQL aging](sbc-data-retention.md). For **HA promote**, do not treat a full restore as failover — use [warm sync + EIP promote](sbc-ha-promote.md) (`--db-only` onto standby, then fence + move EIP).

!!! note "Architecture"
    Lab SBC and OpenSIPS apt packages are **amd64 (x86_64)**. Do not use an arm64 guest for a full install from `apt.opensips.org` on Ubuntu 24.04 today.

## What is in a backup

Local file: `/var/lib/pbx3sbc/bkup/sbcbak.{epoch}.zip`

| Included | Not included (reinstall / ops) |
|----------|--------------------------------|
| MariaDB `opensips` dump (config, peers, users, ops tables) | Hot soft-state (`dialog`, `location`, sessions/cache/jobs) |
| `/etc/opensips/opensips.cfg` + `.mysql_credentials` | Packages, nginx site, LE certs |
| Admin `.env` (APP_KEY + DB) | `/etc/sudoers.d/pbx3sbc-admin` (Fail2ban panels) |

## S3 layout

```text
s3://{org}/sbc/{PBX3_SBC_ID}/backups/{UTC_stamp}/
  backup.zip
  manifest.json
```

Lab: `s3://08jzwn-pbx3/sbc/sbc/backups/`. Objects tagged `class=backup`.

## Filament (VIP / in-service member)

On `https://sbc.pbx3.com/admin` → **System → Backup**:

- **Backup now** (optional Upload to S3) — disabled on standby (host does not hold `advertised_address` / EIP).
- Lists local `sbcbak.*.zip` (Created UTC, Archive ID, size).
- Restore is **not** in the panel — use CLI below on a cold/scratch host.
- After deploying scripts: `sudo ./scripts/setup-admin-panel-sudoers.sh` (adds `sbc-backup-panel.sh`).

Confirm SBC cron + S3: `ls /etc/cron.d/pbx3sbc-backup` and `aws s3 ls s3://08jzwn-pbx3/sbc/sbc/backups/`.

## Warm sync (Fleet — not cold restore)

Fleet → **Edge HA → Sync now**: active uploads a stamp; standby pulls and `restore --db-only`. Daily backstop on control: `pbx3-edge-warm-sync.timer`. See [SBC HA promote](sbc-ha-promote.md) Phase B.

## Create and upload (CLI on a live SBC)

Needs `/etc/pbx3sbc/log-ship.env` (`PBX3_ORG_BUCKET`, `PBX3_SBC_ID`) and IAM for `sbc/{id}/*`.

```bash
sudo /path/to/pbx3sbc/scripts/backup-sbc.sh --trigger manual --upload
sudo /path/to/pbx3sbc/scripts/backup-sbc.sh --dry-run
```

Cron: `pbx3sbc/scripts/cron.d/pbx3sbc-backup.example` → `/etc/cron.d/pbx3sbc-backup`. Local FIFO keeps **9** zips; S3 lifecycle often **30** days (`apply-backup-lifecycle-rule.sh` from Mac ops).

## Fetch from S3 (ops laptop)

```bash
export PBX3_ORG_BUCKET=08jzwn-pbx3
cd /path/to/pbx3   # clone with pbx3-directory/
./pbx3-directory/tools/fetch-latest-sbc-backup.sh --sbc-id sbc --output-dir /tmp
# → /tmp/sbcbak.{epoch}.zip
```

Optional: `--stamp YYYYMMDDThhmmssZ` for a known archive.

---

## Scratch restore runbook (blank host)

End-to-end: new **amd64** Ubuntu 24.04 → install both products → restore from S3 → Filament smoke.  
Do **not** point production VIP / DNS at the scratch host.

For **greenfield only** (no restore), see [Install SBC edge](install-sbc.md).

### 0. Prerequisites

- Host: **x86_64** Ubuntu 24.04 (LAN VM, spare AMD box, or `t3.*` EC2 — **not** `t4g` unless you build OpenSIPS yourself)
- SSH with sudo
- From your laptop: AWS creds that can read `s3://08jzwn-pbx3/sbc/sbc/backups/` (or scp a zip you already fetched)
- Network access to clone public **`pbx3-oss/pbx3sbc`** and **`pbx3sbc-admin`** (HTTPS; no GitHub login required)

### 1. Confirm a backup exists

```bash
aws s3 ls s3://08jzwn-pbx3/sbc/sbc/backups/
```

Fetch (or use a known stamp):

```bash
export PBX3_ORG_BUCKET=08jzwn-pbx3
./pbx3-directory/tools/fetch-latest-sbc-backup.sh --sbc-id sbc --output-dir /tmp
scp /tmp/sbcbak.*.zip user@scratch:/tmp/
```

### 2. Install the edge (`pbx3sbc`)

On the scratch host:

```bash
cd ~
git clone https://github.com/pbx3-oss/pbx3sbc.git
cd pbx3sbc
sudo ./install.sh --advertised-ip <SCRATCH_IP> --preferlan --db-password '<TEMP_DB_PASS>'
```

Use a temporary MariaDB password for install; restore will bring lab credentials and you will realign in step 5.

Check: `systemctl is-active opensips mariadb` → both `active`.

### 3. Install the admin (`pbx3sbc-admin`)

```bash
cd ~
git clone https://github.com/pbx3-oss/pbx3sbc-admin.git
cd pbx3sbc-admin
sudo ./install.sh \
  --db-host localhost \
  --db-name opensips \
  --db-user opensips \
  --db-password '<TEMP_DB_PASS>' \
  --skip-migrations \
  --no-admin-user \
  --server-name <SCRATCH_IP>
```

`--skip-migrations` avoids “table already exists” when you restore next. Empty DB is fine.

### 4. Overlay restore scripts (if not yet on `main`)

Until backup/restore scripts are on the published `main` tip, copy from your workstation:

```bash
scp scripts/backup-sbc.sh scripts/restore-sbc-backup.sh scripts/upload-sbc-backup.sh \
  user@scratch:~/pbx3sbc/scripts/
ssh user@scratch 'chmod +x ~/pbx3sbc/scripts/*sbc*.sh'
```

### 5. Restore

```bash
sudo mkdir -p /var/lib/pbx3sbc/bkup
sudo cp /tmp/sbcbak.*.zip /var/lib/pbx3sbc/bkup/
cd ~/pbx3sbc

sudo ./scripts/restore-sbc-backup.sh --dry-run /var/lib/pbx3sbc/bkup/sbcbak.*.zip

sudo PBX3SBC_ADMIN_ENV=$HOME/pbx3sbc-admin/.env \
  ./scripts/restore-sbc-backup.sh --full --yes --restart \
  /var/lib/pbx3sbc/bkup/sbcbak.*.zip
```

**Never** run `init-database.sh` after this.

OpenSIPS may fail to start until step 6 (lab DB password / public `advertised_address` in the restored cfg).

### 6. Host alignment (required when scratch ≠ original SBC)

```bash
# 6a — MariaDB password = restored credentials
set -a && source /etc/opensips/.mysql_credentials && set +a
sudo mysql -e "ALTER USER 'opensips'@'localhost' IDENTIFIED BY '${DB_PASS}'; FLUSH PRIVILEGES;"

# 6b — advertised_address = this host
sudo sed -i 's/advertised_address="[^"]*"/advertised_address="<SCRATCH_IP>"/' /etc/opensips/opensips.cfg
sudo systemctl reset-failed opensips
sudo systemctl start opensips
systemctl is-active opensips   # expect: active

# 6c — www-data can read admin .env
sudo chmod 755 "$HOME" "$HOME/pbx3sbc-admin"
sudo chown "$(whoami)":www-data "$HOME/pbx3sbc-admin/.env"
sudo chmod 640 "$HOME/pbx3sbc-admin/.env"
# Optional local URL:
sudo sed -i 's|^APP_URL=.*|APP_URL=http://<SCRATCH_IP>|' "$HOME/pbx3sbc-admin/.env"
cd "$HOME/pbx3sbc-admin" && sudo -u www-data php artisan config:clear

# 6d — Fail2ban Log / Status panels (not in the zip)
cd ~/pbx3sbc
sudo ./scripts/setup-admin-panel-sudoers.sh
```

If Filament still 500s on migrations, mark them applied (tables came from the dump):

```bash
cd ~/pbx3sbc-admin
# insert migration basenames into opensips.migrations if migrate:status shows Pending
sudo -u www-data php artisan migrate:status
```

### 7. Smoke

```bash
# Counts should match the source SBC (lab example: users / domains / gateways)
sudo mysql -N -e "
  SELECT COUNT(*) FROM opensips.users;
  SELECT COUNT(*) FROM opensips.domain;
  SELECT COUNT(*) FROM opensips.dr_gateways;
"

curl -sI http://127.0.0.1/admin/login | head -5
```

Browser: `http://<SCRATCH_IP>/admin/login` → Sign in (restored user). Spot-check **Peers**, **Domains**, Fail2ban Log.  
SIP to carriers is optional on a LAN scratch box.

If the login password is unknown, reset hashes via artisan/`tinker` on the scratch host only.

### 8. Tear down

Power off / terminate the scratch host when done. Do not leave it on a public IP with restored production secrets without intent.

---

## Related

- [Backups in the org bucket](backups-org-bucket.md) — **instance** (PBX) backups
- [Rebuild a fleet node from S3](rebuild-from-s3.md) — instance rebuild
- [SBC data retention](sbc-data-retention.md) — MySQL aging / purge (not DR)
