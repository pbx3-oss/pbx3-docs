# Install the Lab control VM

**Audience:** a MS or MacOS tech who can create Ubuntu VMs and paste a few commands.

!!! tip "Operator worksheet"
    Print or copy **[Lab fleet install — operator worksheet](install-lab-worksheet.md)** into your own spreadsheet before you start. It lists every IP, password, token, and minted id across control → SBC → home → SPA → adopt. 

## What we will install in this section. 

The Gatekeeper and object store for the PBX Fleet.  In this exercise we will use the 'Garage' S3 emulator as the store.  

This is **VM A** in Lab deployment: Gatekeeper + Garage (org catalog). It does **not** install the home PBX or the SBC. The Lab walk’s goal is two phones and a call — next page is the SBC, then the home PBX.

If you just want to build a solo PBX (one box, no fleet) go to [Install pbx3 and pbx3api](install-pbx3-pbx3api.md) — no control host, no Garage.

## What you need

| Have it? | Thing |
|:--------:|-------|
| ☐ | Ubuntu **24.04** VM with sudo and SSH |
| ☐ | This VM’s LAN IP (example lab: `192.168.1.33`) |
| ☐ | HTTPS to GitHub. Control installer is `pbx3-directory/` in **pbx3** (public clone when that repo is public; otherwise copy the tree) |
| ☐ | A fleet admin email + password you invent (**10+** characters) |

To begin, bring up a fresh, server-only Ubuntu 24.04 using your hypervisor of choice (Proxmox, Parallels, whatever you have). Make sure it is connected to the same subnet as your laptop/PC to avoid having to route to it. 

You will **not** need AWS or Garage keys.   

## On the control VM


Paste this (creates `~/pbx3/pbx3-directory` and runs the installer):

```bash
sudo apt-get update && sudo apt upgrade
sudo apt-get install -y git
cd ~
git clone --depth 1 https://github.com/pbx3-oss/pbx3.git
cd pbx3/pbx3-directory
sudo ./tools/install-control-host.sh
```

If clone asks for a GitHub login, **pbx3** is still private. Copy the tree from a machine that already has it so this VM has `~/pbx3/pbx3-directory`, then run the installer from there.

Answer:

| Prompt | Example |
|--------|---------|
| Fleet slug | `lab` (bucket becomes `lab-pbx3`) |
| This VM LAN IP | `192.168.1.33` (installer usually detects it) |
| Fleet admin email | your address |
| Fleet admin password | 10+ characters |
| SBC admin API URL | `http://192.168.1.85/api` (use the reserved SBC LAN IP even if that VM is not installed yet) |
| Enable recordings S3 offload? | **Press Enter for No** (default). Yes only if you want a dedicated recordings Garage bucket + home upload plumbing. Tenant sites still opt in with **`rec_s3`**. See [Recordings S3 offload](../fleet/recordings-s3-offload.md). |

When it finishes, open in a browser:

- Health: `http://192.168.1.33/health` → `{"status":"ok"}`
- Catalog: `http://192.168.1.33/catalog/instance-index.json` → `"instances": []` until you adopt a home

The installer also enables **`pbx3-fleet-probe.timer`** (home `/up` every minute so Fleet stays **Healthy** after adopt). Keys stay on the box. The finish banner shows how to **`grep`** **`PBX3_FLEET_SERVICE_TOKEN`** once — reuse that exact value on SBC admin and home install (same as cloud).

The SPA **Fleet** mode sub-app will talk to `http://192.168.1.33` (your control VM address if different) with the email/password you set.

Re-run is safe with the same answers (it will not rotate tokens).

That's it for this VM for now.

## Next

[Install the Lab SBC](install-lab-sbc.md) on an **amd64** VM (then the home PBX).
