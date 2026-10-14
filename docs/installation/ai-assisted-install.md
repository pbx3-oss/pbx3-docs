# AI-assisted install

Paste one of these **kickoff prompts** into your AI coding agent. Replace non-secret `{placeholders}`. Set passwords and keys in your **shell** (or SSH agent); tell the agent the **variable names** only — do not paste secret values into the chat (third-party logs are outside our control).

The agent must follow the linked MkDocs page and **shipped installers** — not invent a new procedure. This is co-pilot + human gates, not unattended install. Prefer **one phase at a time** (stop after the phase’s check); do not paste “do all steps” unless you will watch every command.

Posture: [AI-assisted operations](../getting-started/ai-assisted.md) · lock: `pbx3/workingdocs/AI_ASSISTED_OPERATOR_REQUIREMENTS.md`.

## Human gates (all installs)

Ask before: launching/terminating VMs · DNS cutover · production Let’s Encrypt · IAM / long-lived keys · destructive DB/tenant wipes · unpaid spend beyond what I already approved · live Peer / dialplan / GenAst changes off the documented path.

Do **not** put ops AWS keys or deploy credentials into the admin SPA, public issues, or the chat.

---

## Solo / public node (pbx3 + pbx3api)

Canonical: [Install pbx3 and pbx3api](install-pbx3-pbx3api.md).

```text
Install a solo PBX3 node (pbx3 + pbx3api + TLS) on Ubuntu 24.04.
Follow the MkDocs page: installation/install-pbx3-pbx3api.md (same stack as install-home-host.sh).
Do not invent a parallel installer — use the documented scripts and worksheet.

Worksheet (fill these):
- SSH: {user}@{host}  (key: {key_path_or_agent})
- LE_EMAIL: {email}
- SITE_NAME: {friendly name}
- DOMAIN_TLD: {apex e.g. example.com}
- ADMIN_EMAIL / ADMIN_PASSWORD: set in your shell as env vars; do not paste password values into this chat
- Constraint: follow that MkDocs page and shipped scripts only — no parallel installer.

Ask before: creating/destroying the VM, DNS A-record cutover, production LE cert requests.
Skip fleet service token on this page (solo / commission later).
Done when: https://{instance-fqdn}:44300/up returns 200 without -k, and I can sign in per getting-started/sign-in.md.
```

After install: [Solo trial](../getting-started/solo-trial.md). Fleet later: [Commission](../fleet/commission-instance.md).

---

## Lab home (LAN, no public DNS / LE)

Canonical: [Lab home PBX](install-lab-home.md) · worksheet: [Lab install worksheet](install-lab-worksheet.md).

```text
Bring up Lab home PBX3 on the LAN (no public DNS / LE unless I say so).
Follow: installation/install-lab-home.md and installation/install-lab-worksheet.md.
Use install-home-host.sh / Lab scripts from the pbx3 tree — do not invent steps.

Worksheet:
- SSH: {user}@{lab-host}
- SITE_NAME / admin credentials: {…}
- Any Lab-specific env from the worksheet: {…}

Ask before: wiping an existing Lab DB, changing golden/Lab IPs I did not list.
Done when: API /up (or Lab equivalent in the doc) is healthy and **https://app.pbx3.com** (or Lab Vite) can sign in to this node.
```

---

## Lab SBC (SIP edge)

Canonical: [Lab SBC](install-lab-sbc.md) · also [Install SBC (fleet)](../fleet/install-sbc.md).

```text
Install / configure the Lab SIP edge (pbx3sbc + pbx3sbc-admin) per MkDocs.
Follow: installation/install-lab-sbc.md (Lab) or fleet/install-sbc.md (fleet-shaped).
Use shipped scripts from pbx3sbc / pbx3sbc-admin — do not hand-roll OpenSIPS config.

Worksheet:
- SSH: {user}@{sbc-host}
- Advertised IP / FQDN: {…}
- Admin email/password for Filament: {…}
- LE email if requesting certs: {…}

Ask before: binding VIP / EIP, production LE, wiping opensips DB, changing live Peer/drouting on a production edge.
Done when: admin UI reachable and doc’s smoke checks (e.g. opensips running, admin login) pass.
```

---

## Lab control host (Gatekeeper)

Canonical: [Lab control host](install-lab-control.md).

```text
Install Lab control host (Gatekeeper + catalog/S3-compatible) per installation/install-lab-control.md.
Use documented scripts only.

Worksheet: SSH {…}, bucket/region or Garage local settings {…}, admin {…}.
Ask before: destroying catalog buckets, rotating prod-like IAM, exposing control on the public internet beyond what the doc describes.
Done when: control URL from the doc responds and catalog health checks in that page pass.
```

---

## Fleet onboard / rebuild from S3

Use the dedicated page: [Agent-assisted onboard / rebuild](../fleet/agent-assisted.md) (interim until orchestrated S10.7 / S8.9).
