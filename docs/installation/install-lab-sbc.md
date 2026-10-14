# Install the Lab SBC (SIP edge)

**Audience:** a MS or MacOS tech who can create Ubuntu VMs and paste a few commands.

## What we will install in this section.

This is **VM C**, the SBC Edge router — OpenSIPS + Filament admin on **amd64**. Control (`.33`) is already up. The home PBX comes **after** this page so the home installer can set Egress in one pass.

!!! note "What “SBC” means here"
    Signaling border / proxy-registrar — phones and carriers hit this box for SIP; **RTP stays on the home** (not a media B2BUA). Same idea as the cloud [Install SBC edge](../fleet/install-sbc.md) page.

The PBX3 SBC edge router uses OpenSIPS.  OpenSIPS packages are **x86_64 only**. Do not use this section on ARM VMs.  If you have no option then you will need to install OpenSIPS from source and compile it.   You can find full instructions on the opensips website.

## What you need

| Have it? | Thing |
|:--------:|-------|
| ☐ | Ubuntu **24.04 amd64** VM with sudo and SSH (example lab: `192.168.1.85`) |
| ☐ | HTTPS to GitHub (public **pbx3sbc** + **pbx3sbc-admin**) |
| ☐ | A Filament admin email + password you invent (**10+** characters) — the installer will ask; do not put them in the paste |
| ☐ | **Fleet service token** from control — copy once from the [control installer](install-lab-control.md) finish banner (`grep` on `/etc/pbx3-gatekeeper/.env`). **Reuse the same value** on the home install. **Provision edge** is later, after you adopt the home. |

No Let’s Encrypt on this Lab path (HTTP on the LAN IP).

## 1. Edge (OpenSIPS)

On the SBC VM:

```bash
sudo apt-get update
sudo apt-get install -y git
cd ~
git clone --depth 1 https://github.com/pbx3-oss/pbx3sbc.git
cd pbx3sbc
sudo ./install.sh --advertised-ip 192.168.1.85 --preferlan
```

When asked **Reinitialize database? (y/N):** answer **`y`** on a fresh VM. Invent a DB password and keep it for the admin installer.

Check: `systemctl is-active opensips mariadb` → both `active`.

## 2. Admin (Filament)

**Stop — do not paste passwords or the fleet token in docs.** The block below is safe to copy as-is. The installer will **ask** for:

1. The OpenSIPS **database password** you invented in step 1 (typed twice)
2. A **Filament admin email** (a real address, not a docs hint)
3. A **Filament admin password** (10+ characters, typed twice)
4. The **fleet service token** from control — paste that **exact** value (same as cloud commission). Do not invent a second token. **Provision edge** fails with `401 Unauthorized` if this does not match control (and the home).
5. **Site timezone** for Home/CDR day buckets (IANA, e.g. `America/New_York`). Default = this host’s `/etc/timezone` — press Enter to accept. Does **not** change the OS clock (`timedatectl`). Override with `--site-timezone` or `PBX3_SBC_SITE_TIMEZONE`. Federated preference: keep the host on **UTC**; set site TZ only when operators need a local “today” bucket.

Non-interactive (same as cloud — export once on your ops machine):

```bash
export PBX3_FLEET_SERVICE_TOKEN='paste-from-control-grep'
sudo ./install.sh \
  --server-name 192.168.1.85 \
  --fleet-service-token "$PBX3_FLEET_SERVICE_TOKEN" \
  --site-timezone America/New_York \
  --db-host localhost --db-name opensips --db-user opensips \
  --opensips-mi-url http://127.0.0.1:8888/mi \
  --admin-name Admin
```

If `~/pbx3sbc-admin` already exists, skip `git clone` and only `cd ~/pbx3sbc-admin`.

```bash
cd ~
git clone --depth 1 https://github.com/pbx3-oss/pbx3sbc-admin.git
cd ~/pbx3sbc-admin
sudo ./install.sh \
  --server-name 192.168.1.85 \
  --db-host localhost --db-name opensips --db-user opensips \
  --opensips-mi-url http://127.0.0.1:8888/mi \
  --admin-name Admin
```

`--server-name` is this VM’s LAN IP (no public DNS). Skip `--letsencrypt`.

Open **http://192.168.1.85/admin** (login with the Filament email/password you typed). Do not use snakeoil HTTPS.

Standalone SBC (no fleet): pass **`--skip-fleet-token`**.

## 3. Control already knows this IP

The [control installer](install-lab-control.md) prompt **SBC admin API URL** should already be `http://192.168.1.85/api`. If you pressed Enter earlier, re-run on the control VM (tokens are not rotated):

```bash
cd ~/pbx3/pbx3-directory
sudo PBX3_SBC_ADMIN_API_URL=http://192.168.1.85/api ./tools/install-control-host.sh
```

## 4. WebRTC / WSS on the Lab SBC (required for SPA Line test)

Desk SIP on **UDP 5060** works after steps 1–2. **Browser WebRTC does not** until you enable **WSS on the edge**.

OpenSIPS ships with the W1 WSS block **commented out** (default off). Lab installs also skip Let’s Encrypt, so use the lab helper (self-signed cert for the SBC LAN IP). It installs WSS/TLS modules with **`force-confold`** (so an opensips package bump does not clobber your cfg) and enables **exactly one** cert pair (enabling both template examples makes OpenSIPS fail to start).

On the **SBC VM** (adjust IP if yours is not `.85`):

```bash
cd ~/pbx3sbc
git pull --ff-only   # need scripts/enable-lab-wss.sh
sudo SBC_IP=192.168.1.85 ./scripts/enable-lab-wss.sh
```

Check: `ss -lnt | grep 8089` and `ss -ulnt | grep 5060` both show listeners; desk phones re-REGISTER.

**SPA Line test** (after [Lab SPA](install-lab-spa.md) + a WebRTC extension): open the extension → **Line test** and set:

| Field | Lab value |
|-------|-----------|
| WSS URL | `wss://192.168.1.85:8089/ws` (not the default `sbc.pbx3.com`) |
| SIP domain | tenant FQDN (e.g. `nqybwn.pbx3.com`) |
| SIP user | extension **shortuid** (not `105`) |

!!! warning "Trust the self-signed cert"
    Chrome/Safari reject untrusted `wss://` handshakes. On the Mac, copy `/etc/opensips/tls/lab-sbc-fullchain.pem` off the SBC into **Keychain Access → System** and set **Always Trust** for SSL, or use [mkcert](https://github.com/FiloSottile/mkcert) for `192.168.1.85`. Cloud fleets use real Let’s Encrypt — see [Install SBC](../fleet/install-sbc.md) § WebRTC WSS.

Cloud VIP path (LE + `setup-opensips-wss.sh`, then enable **one** cert pair by hand): **`pbx3sbc/workingdocs/~/GiT/pbx3-ops/devdocs/oss-move/WEBRTC_W1_SBC_LAB.md`**.

That's it for this VM for now. Do **not** Provision edge yet — there is no home in the catalog.

## Next

[Install the Lab home PBX](install-lab-home.md). Use this SBC LAN IP as the home **SBC egress host**. Paste the **same** fleet token again (or reuse `PBX3_FLEET_SERVICE_TOKEN` from your export).
