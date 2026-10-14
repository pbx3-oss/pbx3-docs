# Lab fleet install — operator worksheet

**Audience:** anyone running the [Lab install sequence](install-lab-control.md) (control → SBC → home → SPA → adopt).

Print this page or copy the tables into your own spreadsheet **before** you start. Fill in **Value** as you go. The lab walk mixes values you **invent**, values you **copy once** from control, and values the **installer mints** — easy to lose track on a notepad.

!!! warning "Private — do not commit filled copies"
    This worksheet holds passwords and fleet tokens. Keep filled copies on your machine or password manager only. **Do not** commit them to git, tickets, or shared docs.

**Install sequence:** [Control](install-lab-control.md) → [SBC](install-lab-sbc.md) → [Home](install-lab-home.md) → [Admin SPA (Lab Vite)](install-lab-spa.md) → [Adopt + Provision edge](install-lab-adopt.md). Cloud fleets use **https://app.pbx3.com** instead of Vite.

---

## Build header (fill once)

| Field | Your value |
|-------|------------|
| Build name / label | |
| Date | |
| Operator | |
| Subnet (e.g. `192.168.1.0/24`) | |
| Hypervisor / notes | |

---

## A. Plan before VM install (reserve these)

Decide LAN IPs up front so the [control installer](install-lab-control.md) can take the SBC URL before the SBC exists.

| Field | Your value | Notes |
|-------|------------|-------|
| Control VM LAN IP | | Example `192.168.1.33` |
| Home VM LAN IP | | Example `192.168.1.31` — any Ubuntu 24.04 arch |
| SBC VM LAN IP | | Example `192.168.1.85` — **amd64 only** |
| Fleet slug | | Becomes org bucket `{slug}-pbx3` (e.g. `lab` → `lab-pbx3`) |
| SBC admin API URL (for control prompt) | | `http://<SBC-IP>/api` |

---

## B. Control VM — [install-lab-control](install-lab-control.md)

### You invent (installer prompts)

| Field | Your value | Rule |
|-------|------------|------|
| Fleet admin email | | Real address — **Fleet console** login |
| Fleet admin password | | Min **10** characters |

### Copied or read after install (minted on host — copy once)

Run on the **control** VM when the installer finishes:

```bash
sudo grep -E '^(PBX3_ORG_BUCKET|PBX3_FLEET_SERVICE_TOKEN)=' /etc/pbx3-gatekeeper/.env
```

| Field | Your value | Reused at |
|-------|------------|-----------|
| Org bucket (`PBX3_ORG_BUCKET`) | | Home installer |
| Fleet service token (`PBX3_FLEET_SERVICE_TOKEN`) | | **SBC admin + home** — same exact value; never mint a second token |
| Control health URL | `http://<control-IP>/health` | Sanity check → `{"status":"ok"}` |
| Catalog URL | `http://<control-IP>/catalog/instance-index.json` | Empty `instances` until adopt |

### Ops-only (stay on control — note location, do not paste into shared docs)

| Field | Notes |
|-------|-------|
| `GATEKEEPER_API_TOKEN` | Break-glass API bearer — `/etc/pbx3-gatekeeper/.env` |
| Garage / S3 keys | Installer-generated — same `.env` file |

---

## C. SBC VM — [install-lab-sbc](install-lab-sbc.md)

### You invent

| Field | Your value | Rule |
|-------|------------|------|
| OpenSIPS / MariaDB password | | Step 1 edge install — **reused** in Filament admin install |
| Filament admin email | | **SBC Filament** login (`http://<SBC-IP>/admin`) |
| Filament admin password | | Min **10** characters |

### Copied from control (section B)

| Field | Your value |
|-------|------------|
| Fleet service token | *(same as control — paste again or reuse `export PBX3_FLEET_SERVICE_TOKEN=…`)* |

### Derived

| Field | Your value |
|-------|------------|
| SBC LAN IP | |
| Edge `--advertised-ip` | Same as SBC LAN IP (lab) |
| Filament URL | `http://<SBC-IP>/admin` |

---

## D. Home VM — [install-lab-home](install-lab-home.md)

### You invent (installer prompts)

| Field | Your value | Rule |
|-------|------------|------|
| Domain apex | | Usually **Enter** → default `pbx3.com` |
| Site name (friendly Name) | | e.g. `Lab Home` — Register instance **Label** |
| Instance admin SPA email | | **Instance admin** login — not fleet email |
| Instance admin SPA password | | Min **8** characters |

### Copied from control (section B)

| Field | Your value |
|-------|------------|
| Fleet service token | *(same as B + C)* |
| Org bucket | *(same as B)* |

### From SBC plan (section A / C)

| Field | Your value |
|-------|------------|
| SBC egress host | SBC LAN IP — **do not skip** on fleet lab home |

### Minted by installer (record from finish banner or SQL)

On the **home** VM after install:

```bash
sudo sqlite3 /opt/pbx3/db/sqlite.db \
  "SELECT id, shortuid, fqdn, sitename FROM globals WHERE pkey='global';"
```

| Field | Your value | Used at |
|-------|------------|---------|
| Instance id (KSUID) | | Fleet → Register instance |
| shortuid | 6 chars | Reference / SIP auth context |
| FQDN | `{shortuid}.pbx3.com` | Register instance |
| sitename | | Register instance **Label** |
| Home API base (adopt) | `https://<home-IP>:44300/api` | Register instance |
| Home `/up` | `curl -k https://<home-IP>:44300/up` → **200** | Pre-adopt check |

---

## E. Your PC — [install-lab-spa](install-lab-spa.md)

Create **`pbx3spa/.env.development`** (not in git). Adjust IPs to match sections A and D.

| Variable | Your value |
|----------|------------|
| `VITE_API_PROXY_TARGET` | `https://<home-IP>:44300` |
| `VITE_DEFAULT_API_BASE_URL` | `http://localhost:5173/api` |
| `VITE_CATALOG_PROXY_TARGET` | `http://<control-IP>` |
| `VITE_INSTANCE_DIRECTORY_URL` | `/dev-catalog/catalog/instance-index.json` |
| `VITE_FLEET_GATEKEEPER_PROXY_TARGET` | `http://<control-IP>` |
| `VITE_FLEET_GATEKEEPER_URL` | `/fleet-gk` |
| Browser URL | `http://localhost:5173` |

### Two logins — do not mix

| Mode | Email (from) | Password (from) |
|------|----------------|-----------------|
| **Fleet console** | Section B | Section B |
| **Instance admin** | Section D | Section D |

---

## F. Adopt + Provision edge — [install-lab-adopt](install-lab-adopt.md)

| Step | Done ☐ | Value / notes |
|------|--------|---------------|
| Control `/health` 200 | ☐ | |
| Home `/up` 200 | ☐ | |
| SBC Filament login OK | ☐ | |
| SPA → **Fleet console** login | ☐ | Section B creds |
| **Register instance** — KSUID | ☐ | Section D |
| **Register instance** — FQDN | ☐ | Section D |
| **Register instance** — API base | ☐ | Section D |
| **Register instance** — Label | ☐ | sitename |
| **Register instance** — Environment | ☐ | `lab` |
| Catalog lists home row | ☐ | Refresh catalog if missing |
| Instance picker → **home** login | ☐ | Section D creds |
| **Provision edge** on instance row | ☐ | Needs matching fleet token on SBC + home |

Do **not** run Mac `onboard-fleet-instance.sh` on the Lab happy path.

---

## G. Extensions and phones

Create tenant + extensions in **instance admin** (not fleet). **Save**, then **Commit** before aiming phones at the SBC.

| Extension | Number | SIP password (shown once on create/regen) | Phone SIP server |
|-----------|--------|-------------------------------------------|------------------|
| Phone 1 | e.g. **101** | | **SBC LAN IP** — not home |
| Phone 2 | e.g. **102** | | **SBC LAN IP** — not home |

**Order:** extensions → Commit → point phones at SBC → call 101 ↔ 102. CAGI must be on the home ([ARM compile](install-lab-home.md#cagi-needed-for-a-call)).

---

## H. Verification checklist

| Check | Expected | Done ☐ |
|-------|----------|--------|
| Control `/health` | HTTP 200 | ☐ |
| Catalog after register | Home instance row | ☐ |
| Home `fleet-posture` | `"fleet": true` in SPA DevTools | ☐ |
| Egress trunk on home | SQL `trunks` row `Egress` active | ☐ |
| SBC `/admin` login | Filament OK | ☐ |
| Provision edge | No 401 on fleet token mismatch | ☐ |
| 101 ↔ 102 call | Audio both ways | ☐ |

---

## Spreadsheet import (optional)

Copy any table into Excel, Numbers, or Google Sheets. Suggested extra columns if you rebuild as a sheet:

| Phase | VM | Field | Source (invent / copy / mint) | Value | Used again at |

**Source legend:** **invent** = you choose before/during prompts; **copy** = from control grep once; **mint** = installer/SQL output you record after the step.

---

## Non-interactive tip (one export, zero VM pastes)

On your ops machine after control install:

```bash
export PBX3_FLEET_SERVICE_TOKEN='paste-from-control-grep'
export PBX3_ORG_BUCKET='lab-pbx3'   # match your slug
```

Use the same exports for [SBC admin](install-lab-sbc.md) (`--fleet-service-token`) and [home](install-lab-home.md) installers. Same path as cloud commission.

---

## Next

Start the walk: [Install the Lab control VM](install-lab-control.md).
