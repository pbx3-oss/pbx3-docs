# Install the Lab home PBX

**Audience:** a MS or MacOS tech who can create Ubuntu VMs and paste a few commands. 

## What we will install in this section.  

This is **VM B** in Lab deployment: pbx3 + pbx3api (Asterisk). No public DNS, no Let’s Encrypt. SIP to the phones goes through the SBC you just installed.

The [cloud/solo install](install-pbx3-pbx3api.md) page uses the same `install-home-host.sh` stack, then DNS and Let’s Encrypt. Use **this** page on a LAN VM (no public DNS/LE; fleet token at install).

Already done: [control](install-lab-control.md), then [SBC](install-lab-sbc.md).

## What you need

| Have it? | Thing |
|:--------:|-------|
| ☐ | Ubuntu **24.04** VM with sudo and SSH |
| ☐ | This VM’s LAN IP (example lab: `192.168.1.31`) |
| ☐ | HTTPS to GitHub (**pbx3api** is public; **pbx3** copy the tree if clone asks for login) |
| ☐ | A site **Name** + admin email/password you invent (**8+** characters; not a docs placeholder like `admin@example.com`) |

To begin, bring up a fresh, server-only Ubuntu 24.04 using your hypervisor of choice (Proxmox, Parallels, whatever you have). Make sure it is connected to the same subnet as your laptop/PC to avoid having to route to it (e.g. 192.168.0.0/16, 172.16.0.0/12, 10.0.0.0/8). 

A call needs **CAGI** on this box. The home installer can skip it (`PBX3_INSTALL_CAGI` unset) because **pbx3cagi** is still private. Compile on the guest after the installer (do not copy a MacOS binary) — commands are at the end of this page.

## On the home VM

Paste this (public **pbx3** + **pbx3api**; run the installer from the pbx3 tree so the `.deb` on `main` is found):

```bash
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install -y git
cd ~
git clone --depth 1 https://github.com/pbx3-oss/pbx3.git
git clone --depth 1 https://github.com/pbx3-oss/pbx3api.git pbx3/pbx3api
cd pbx3
sudo ./scripts/install-home-host.sh
```

**pbx3api** is public. If the **pbx3** clone asks for a GitHub login, copy `pbx3/` (including `scripts/install-home-host.sh` and the `pbx3_*.deb` on `main`) from a machine that already has it, clone **pbx3api** into `pbx3/pbx3api`, then run the installer.

Answer:

| Prompt | Example |
|--------|---------|
| Domain apex | **Press Enter** (default `pbx3.com`). Type nothing. You are **not** asked for an instance FQDN — it is minted as `{shortuid}.pbx3.com`. |
| Site name | `Lab Home` |
| Admin SPA email | your address |
| Admin SPA password | 8+ characters |
| Fleet service token | same value as control / SBC (`PBX3_FLEET_SERVICE_TOKEN` from control `grep`) |
| Org bucket | `lab-pbx3` |
| SBC egress host | SBC LAN IP (example `192.168.1.85`) — **do not skip** on a fleet lab home |

When a fleet token is provided, the installer writes fleet API settings to **`/opt/pbx3api/.env`** (`PBX3_FLEET_MODE`, org bucket, egress host), seeds the instance-wide **`Egress`** trunk when the seeder is present, and runs **`link-asterisk-configs.sh`** so **`/etc/asterisk/pjsip_ready_*.conf`** symlinks exist before your first Commit.

The seeder is shipped at:

1. `pbx3/scripts/seed-fleet-egress-trunk.sh` — public clone / git tip
2. `/opt/pbx3/scripts/seed-fleet-egress-trunk.sh` — from the **pbx3** deb
3. `pbx3/pbx3-directory/tools/seed-fleet-egress-trunk.sh` — full monorepo checkout

Non-interactive example (recommended — same as cloud; export token once):

```bash
export PBX3_FLEET_SERVICE_TOKEN='paste-from-control-grep'
sudo PBX3_FLEET_SERVICE_TOKEN="$PBX3_FLEET_SERVICE_TOKEN" PBX3_ORG_BUCKET=lab-pbx3 \
     PBX3_SBC_EGRESS_HOST=192.168.1.85 ./scripts/install-home-host.sh
```

## Clean re-install (same VM)

Restore a **clean Ubuntu snapshot**, or on the home VM before re-running the installer:

```bash
sudo PBX3_CLEAN_INSTALL=1 \
     PBX3_FLEET_SERVICE_TOKEN='…' PBX3_ORG_BUCKET=lab-pbx3 \
     PBX3_SBC_EGRESS_HOST=192.168.1.85 \
     ./scripts/install-home-host.sh
```

`PBX3_CLEAN_INSTALL=1` removes `/opt/pbx3/db/sqlite.db` and re-provisions a fresh `{shortuid}.pbx3.com` identity. The installer **fails** at the end if fleet `.env`, Egress trunk, or `/etc/asterisk/pjsip_ready_*.conf` symlinks are wrong — fix before adopt/phones.

## Prove it

On the VM (the installer also prints this):

```bash
curl -k -sS -o /dev/null -w "%{http_code}\n" https://127.0.0.1:44300/up
# expect 200
```

UFW leaves **SSH :22** and **API :44300** open from anywhere so install cannot lock you out. After the SPA is up, open **Admin → Firewall**, narrow those Source fields to your LAN/ops CIDR(s), then Save and Apply. See [Firewall](../admin/firewall.md).

## Fleet posture on the home (verify)

Fleet vs singleton is decided by **`GET /api/fleet-posture`** (SPA uses this to hide **Instance admin → Tenants → Create** on fleet nodes).

On the instance, either signal makes **`fleet: true`**:

| Signal | Where |
|--------|--------|
| **`PBX3_FLEET_MODE=true`** | `/opt/pbx3api/.env` (written by home installer when fleet token provided) |
| **Active `Egress` trunk** | SQLite `trunks` row `pkey='Egress'` and `active='YES'` |

Tenant create in Fleet does **not** seed the Egress trunk — that is **once per instance** at home install (or manual recovery below).

After install, on the home VM:

```bash
grep -E '^PBX3_FLEET_MODE=|^PBX3_SBC_EGRESS_HOST=' /opt/pbx3api/.env
sudo sqlite3 /opt/pbx3/db/sqlite.db "SELECT pkey, active, host FROM trunks WHERE pkey='Egress';"
cd /opt/pbx3api && sudo -u www-data php artisan config:clear
```

In the SPA (logged into instance admin), DevTools console:

```javascript
fetch('/api/fleet-posture', { credentials: 'include' }).then(r => r.json()).then(console.log)
```

Expect `"fleet": true`. Then **Tenants** shows **Create via Fleet**, not a blue **Create** button.

## If fleet `.env` or Egress is wrong (manual recovery)

### Fix `/opt/pbx3api/.env`

Fleet keys must be **uncommented**, each on its **own line**. Example:

```env
PBX3_FLEET_MODE=true
PBX3_FLEET_SERVICE_TOKEN=<from control /etc/pbx3-gatekeeper/.env>
PBX3_ORG_BUCKET=lab-pbx3
PBX3_DIRECTORY_BACKUP_UPLOAD=true
PBX3_SBC_EGRESS_HOST=192.168.1.85
```

**Known installer gap (#5h):** if `PBX3_FLEET_MODE` appears glued to a comment line on an **old** install (pre-fix `.env` append), split it onto its own line, then run `php artisan config:clear`. New installs use a fixed append + trailing newline on `.env.example`.

### Seed Egress trunk (if SQL returns no row)

From your **Mac** (adjust paths and IP):

```bash
scp ~/GiT/pbx3-master/pbx3/pbx3-directory/tools/seed-fleet-egress-trunk.sh tech@192.168.1.31:/tmp/
ssh tech@192.168.1.31
chmod +x /tmp/seed-fleet-egress-trunk.sh
sudo PBX3_SBC_EGRESS_HOST=192.168.1.85 /tmp/seed-fleet-egress-trunk.sh /opt/pbx3/db/sqlite.db
sudo /opt/pbx3/scripts/genAst.sh
sudo /opt/pbx3/scripts/link-asterisk-configs.sh
sudo systemctl restart asterisk
```

Re-check `fleet-posture` and the Tenants list.

## CAGI (needed for a call)

**pbx3cagi** is still private — there is nothing to `cd` into until you copy the tree. From the **Mac** (adjust the local path if yours differs):

```bash
rsync -az --exclude .git ~/GiT/pbx3-master/pbx3cagi/ tech@192.168.1.31:~/pbx3cagi/
```

Then on the **home VM**:

```bash
sudo apt-get install -y build-essential libbsd-dev
cd ~/pbx3cagi/pbx3cagi-1.0.0/csource
make clean && make
sudo cp -a pbx3cagi /usr/share/asterisk/agi-bin/pbx3cagi.arm64
sudo chmod 755 /usr/share/asterisk/agi-bin/pbx3cagi.arm64
sudo ln -sfn pbx3cagi.arm64 /usr/share/asterisk/agi-bin/pbx3cagi
file /usr/share/asterisk/agi-bin/pbx3cagi.arm64   # ELF aarch64
```

On **amd64**, clone **pbx3cagi** when it is public and set `PBX3_INSTALL_CAGI=1`, or `apt install` the `_all.deb`.

That's it for this VM for now.

Optional: **recordings → S3** is **off by default** at install. If control enabled capability, home plumbing can be wired and each tenant opts in with **`rec_s3`** (default No) — MkDocs [Recordings S3 offload](../fleet/recordings-s3-offload.md).

## Next

1. [Admin SPA](install-lab-spa.md) — Lab LAN: Vite on your PC (`npm run dev` → http://localhost:5173). Cloud/public fleets use **https://app.pbx3.com** (no local build).
2. [Adopt this home into Fleet](install-lab-adopt.md), then **Provision edge**. Ops Mac onboard is **not** the Lab happy path.
