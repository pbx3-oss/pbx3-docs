# Onboard a second instance

Join a node that **already finished** [Install](../installation/install-pbx3-pbx3api.md) to the org bucket and catalog (adopt only).

For a **new** fleet home: finish [Install](../installation/install-pbx3-pbx3api.md), then **[Commission Step 1](commission-instance.md#step-1--adopt-into-fleet-onboard)** (onboard → edge).

## Before you start

- Node installed; `GET /up` → 200
- Stable `globals.id` / shortuid / fqdn
- No fleet `.env` yet on the node
- Ops Mac AWS identity is **not** a node role (`pbx3-node-*`)
- Fleet service token ready (same as Gatekeeper — see [Commission § Step 1](commission-instance.md#step-1--adopt-into-fleet-onboard))

## Preferred script (**ops Mac only**)

Do **not** run this on the new EC2. Needs a local **`pbx3`** clone (`pbx3-directory/tools/`). If missing on the Mac:

```bash
mkdir -p ~/GiT/pbx3-master && cd ~/GiT/pbx3-master
git clone https://github.com/pbx3-oss/pbx3.git
cd pbx3 && git checkout main && git pull --ff-only
```

```bash
cd ~/GiT/pbx3-master/pbx3/pbx3-directory/tools
export PBX3_ORG_BUCKET=08jzwn-pbx3
./onboard-fleet-instance.sh \
  --instance-id i-XXXXXXXX \
  --ssh ubuntu@NODE.pbx3.com \
  --ssh-key ~/path/to/key.pem \
  --region us-east-1
# dry-run first:
# ./onboard-fleet-instance.sh ... --dry-run
```

## What it covers

IAM policy/role/profile → attach EC2 → SSH `.env` (`PBX3_ORG_BUCKET`, fleet token, backup upload) → seed **Egress** → S3 smoke → register catalog → shared SPA (**[app.pbx3.com](https://app.pbx3.com)**) shows a second row (bucket CORS must allow that origin).

After onboard: Fleet → **Instances** → **Provision edge** (`sip:{PUBLIC_IP}:5060`) and Fail2ban whitelist — see [Commission Step 2](commission-instance.md#step-2--sbc-edge-required-before-fleet-create-tenant).

## Manual debug phases

Same order as the script; stop after any red preflight. On node:

```bash
sudo php artisan pbx3:fleet-preflight
```

## Remove from catalog

Prefer the full teardown when retiring the EC2: **[Decommission a fleet instance](decommission-instance.md)** (tenants → Decom → SBC → DNS → terminate).

Catalog-only (node still running):

```bash
./unregister-instance.sh --id {KSUID}          # soft
./unregister-instance.sh --id {KSUID} --remove # hard catalog row
```

Does not delete S3 backups or stop calls. SPA: Fleet → **Instances** → **Decom**.

Also: [Cloud / S3](../cloud/bucket-layout-cors.md), [Agent-assisted](agent-assisted.md).
