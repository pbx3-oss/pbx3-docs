# Commission a fleet instance

Join a **new home** to an existing fleet after the stack is already installed: onboard → SBC edge → optional tenant smoke.

This creates / uses a **new KSUID** from Install. It is **not** [Rebuild from S3](rebuild-from-s3.md) (same identity after EC2 loss).

| Phase | What | Where |
|-------|------|--------|
| **0** | EC2 → `install-home-host.sh` (skip fleet) → DNS → LE → sign in at **app.pbx3.com** | **[Install pbx3 and pbx3api](../installation/install-pbx3-pbx3api.md)** (required first) |
| **1** | Fleet adopt (IAM, catalog, Egress) | This page |
| **2** | Provision edge + Fail2ban | This page |

EC2, EIP, SG (including **80** for LE and **44300** for ops/Gatekeeper), and DNS are part of Install — do not re-do them here.

!!! tip "Install not done yet?"
    Finish [Install](../installation/install-pbx3-pbx3api.md) first (trusted `/up` + login at [app.pbx3.com](https://app.pbx3.com)).  
    Adopt-only details also on [Onboard](onboard-instance.md).  
    Retire a home → [Decommission](decommission-instance.md).

## Before you start

| Have it? | Thing |
|:--------:|-------|
| ☐ | [Install](../installation/install-pbx3-pbx3api.md) checklist complete on this node |
| ☐ | `INSTANCE_ID` (`i-…`), `PUBLIC_IP`, SSH access + `.pem` |
| ☐ | Worksheet exports from Install: `KSUID`, `SHORTUID`, `INSTANCE_FQDN` |
| ☐ | **Ops AWS identity** on Mac (`aws sts get-caller-identity` — **not** `pbx3-node-*`) |
| ☐ | **`pbx3` git clone on the Mac** with `pbx3-directory/tools/onboard-fleet-instance.sh` (see Step 1b) |
| ☐ | Existing fleet **org bucket** (lab: `08jzwn-pbx3`) |
| ☐ | **Fleet service token** (same as Gatekeeper — see Step 1a) |

### Worksheet

```bash
export AWS_DEFAULT_REGION=us-east-1
export KEY_FILE=/path/to/your.pem
export PBX3_ORG_BUCKET=08jzwn-pbx3
export INSTANCE_ID=i-…
export PUBLIC_IP=…
export SSH_HOST=ubuntu@${PUBLIC_IP}
# or: export SSH_HOST=ubuntu@virginia1.pbx3.com

export KSUID=…
export SHORTUID=…
export INSTANCE_FQDN=…
```

---

## Step 1 — Adopt into fleet (onboard)

| Substep | Where it runs |
|---------|----------------|
| **1a** token | **Ops Mac** (SSH *to* control/sibling only to *read* the secret) |
| **1b** onboard script | **Ops Mac only** — never on the new EC2 |
| **1c** sign-off | **New node** (SSH in) + Fleet mode on **app.pbx3.com** |

The new instance does **not** need a `pbx3` git clone. The Mac does.

### 1a — Fleet service token (ops Mac)

**What it is:** One long-lived shared secret for the whole fleet. Gatekeeper sends it as a bearer token when calling each node’s `POST /api/fleet/*` (create tenant, move, etc.). Onboard writes the **same** value into the new node’s `/opt/pbx3api/.env` as `PBX3_FLEET_SERVICE_TOKEN`.

| It is | It is not |
|-------|-----------|
| Ops secret on **Gatekeeper** (authoritative) and every fleet node | SPA admin email/password |
| Created **once** when the control/Gatekeeper host was first stood up | Something you invent per new home |
| Same string fleet-wide | AWS IAM / instance role |

**Where it comes from:** Whoever bootstrapped Gatekeeper generated it (typically `openssl rand -hex 32`) and put it in **`/etc/pbx3-gatekeeper/.env`** on the control host. You **copy** that value; you do not mint a new one for this node.

**On the ops Mac**, read it (first source that works):

```bash
# 1) Authoritative — control / Gatekeeper host (lab: control.pbx3.com)
ssh ubuntu@control.pbx3.com \
  "grep -E '^PBX3_FLEET_SERVICE_TOKEN=' /etc/pbx3-gatekeeper/.env | head -1"

# 2) Any already-onboarded sibling (golden, etc.) — same string if fleet is healthy
ssh -i "$KEY_FILE" ubuntu@08jzwn.pbx3.com \
  "grep -E '^PBX3_FLEET_SERVICE_TOKEN=' /opt/pbx3api/.env | head -1"
```

Still on the **Mac**, export (value only, after `=`):

```bash
export PBX3_FLEET_SERVICE_TOKEN='paste-value-only'
python3 -c "import os; t=os.environ.get('PBX3_FLEET_SERVICE_TOKEN',''); print('token_len', len(t))"
```

First-ever fleet only (not this path): generate once (`openssl rand -hex 32`), put in Gatekeeper `/etc/pbx3-gatekeeper/.env`, restart Gatekeeper, then use that value on every node forever.

### 1b — Onboard script (**ops Mac only**)

!!! warning "Do not run this on the new EC2"
    `cd …/pbx3-directory/tools` and `./onboard-fleet-instance.sh` run on your **laptop**. The script uses local AWS CLI + SSH *into* the node. Looking for that path on virginia1 / the target will fail — there is no git tree there by design.

The tool ships in the **`pbx3-oss/pbx3`** repo on the Mac:

`pbx3/pbx3-directory/tools/onboard-fleet-instance.sh`

If missing on the Mac (same clone Install used for `.deb`s):

```bash
# ON THE MAC
mkdir -p ~/GiT/pbx3-master
cd ~/GiT/pbx3-master
git clone https://github.com/pbx3-oss/pbx3.git
cd pbx3 && git checkout main && git pull --ff-only

export PBX3_REPO=~/GiT/pbx3-master/pbx3
test -x "${PBX3_REPO}/pbx3-directory/tools/onboard-fleet-instance.sh" && echo "onboard tool ok"
```

Then, still **on the Mac** (token already exported from 1a):

```bash
# ON THE MAC — not on the node
cd "${PBX3_REPO:-$HOME/GiT/pbx3-master/pbx3}/pbx3-directory/tools"
./onboard-fleet-instance.sh \
  --instance-id "$INSTANCE_ID" \
  --ssh "$SSH_HOST" \
  --ssh-key "$KEY_FILE" \
  --region "$AWS_DEFAULT_REGION" \
  --org-bucket "$PBX3_ORG_BUCKET"
```

**What it does:** from the Mac → AWS IAM + SSH to the node → catalog register → fleet `.env` → seed **Egress** → genAst / runLinker / Asterisk restart → S3 smoke.

### 1c — Sign-off (node + Fleet SPA at app.pbx3.com)

**SSH into the new node** for these checks:

```bash
grep -E '^(PBX3_ORG_BUCKET|PBX3_FLEET_MODE|PBX3_FLEET_SERVICE_TOKEN)=' /opt/pbx3api/.env
sqlite3 /opt/pbx3/db/sqlite.db "SELECT pkey, active, host FROM trunks WHERE pkey='Egress';"
curl -sS http://169.254.169.254/latest/meta-data/iam/security-credentials/
cd /opt/pbx3api && sudo php artisan config:clear && sudo php artisan pbx3:fleet-preflight
```

Fleet SPA → **Instances** → refresh — new shortuid / Name should appear.

---

## Step 2 — SBC edge (required before Fleet Create tenant)

Onboard makes the **node** able to dial toward the SBC. It does **not** register the home on the edge.

### 2a — Provision edge (Fleet SPA)

Fleet → **Instances** → row menu → **Provision edge**.

| Field | Value |
|-------|--------|
| Backend SIP URI | **`sip:{PUBLIC_IP}:5060`** — EIP / public IP, **not** the FQDN |

Click **Create edge**. Catalog should show a **Setid**.

!!! warning "FQDN backends"
    The SBC rejects DNS-name backends here. Always **`sip:IP:5060`**.

If Setid is blank after a rebuild but the edge already exists, use **Link setid** (catch-up only).

### 2b — Fail2ban whitelist (manual until #5e)

SBC admin (lab: `https://sbc.pbx3.com/admin`) → **Fail2ban → Whitelist** → add this home’s public IP **`/32`**.

---

## Step 3 — Optional: first tenant and smoke

1. Fleet → **Tenants** → **Create** on this home (setid required).  
2. Allocate / assign DIDs.  
3. Inbound + outbound through the SBC.  
4. Catalog **Reconcile** if domains changed.  
5. Backup → panel shows **local+S3**.

Do **not** invent node-only tenants missing from the catalog.

---

## Checklist

- [ ] [Install](../installation/install-pbx3-pbx3api.md) complete
- [ ] `onboard-fleet-instance.sh` (token + IAM + catalog + Egress)
- [ ] `pbx3:fleet-preflight` green; instance in Fleet picker
- [ ] **Provision edge** `sip:{PUBLIC_IP}:5060`; Setid visible
- [ ] Fail2ban whitelist `/32` for home IP
- [ ] (Optional) tenant / DID / call smoke + S3 backup

## Failure cheat sheet

| Symptom | Likely fix |
|---------|------------|
| Stack / `/up` / LE / hairpin / empty users | [Install failure cheat sheet](../installation/install-pbx3-pbx3api.md#failure-cheat-sheet) |
| Onboard AWS error | Mac using `pbx3-node-*` role — switch to ops identity |
| Preflight S3 red | IAM attach lag; empty `AWS_ACCESS_KEY_*` in `.env` |
| Fleet Create: no setid | Step 2a Provision edge |
| Fleet Create: service token | Step 1a — token must match Gatekeeper |
| Egress Unavail / home banned | Step 2b Fail2ban whitelist |
| Want **same** KSUID as a dead box | [Rebuild from S3](rebuild-from-s3.md) |

## Related

- [Install pbx3 and pbx3api](../installation/install-pbx3-pbx3api.md) — required before this page  
- [Onboard](onboard-instance.md) — same adopt script  
- [Decommission](decommission-instance.md) · [Rebuild from S3](rebuild-from-s3.md) · [Agent-assisted](agent-assisted.md)
