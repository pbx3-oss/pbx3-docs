# Restrict provision HTTPS (edge :41363)

Optional **site egress allowlist** for phone provisioning on the SBC. Complements vendor **mTLS** — it does not replace it.

Filament: **System → Provision access** (sibling of **Management access**, which only covers admin `:443`).

## When to use

| Mode | Use when |
|------|----------|
| **Lockdown off** (default) | Mixed WFH / unknown phone egress; typical cloud RPS until sites are known |
| **Lockdown on** | Hard sites with stable office (or VPN) public egress CIDRs listed |

Phones must **egress from an allowed CIDR**. RPS only discovers the URL; the handset (or site NAT) opens TCP to `provision.{apex}:41363`.

## What it does / does not

- **Does:** UFW allowlist on **`41363/tcp` only** (tagged `pbx3sbc-prov`).
- **Does not:** touch SIP, RTP, WSS, SSH, or admin HTTPS (`:443` — use Management access).
- **Does not:** replace mTLS, MAC map, or `sndcreds`.

## Operator steps

1. Deploy `pbx3sbc` tip (includes `apply-provision-access-ufw.sh`).
2. Re-run `sudo ./scripts/setup-admin-panel-sudoers.sh` and ensure `setup-php-fpm-ufw-write.sh` has run (same as Management access).
3. Open **System → Provision access**, add site CIDRs (comments help), optionally **Add my IP** for laptop curl tests.
4. Turn **Restrict provision HTTPS** on → **Apply**.
5. Confirm status shows tagged allows; test GET from an allowed egress and a denied one.

## Recovery

Lockdown does **not** lock you out of Filament (`:443`). If phones stop fetching configs: turn lockdown off and Apply, or:

```bash
sudo ufw allow 41363/tcp
# or edit state JSON lockdown=false and re-run apply-provision-access-ufw.sh
```

## Related

- [Desk phone RPS enrollment](phone-provisioning-rps.md)
- [Provision streams](phone-provisioning-streams.md)
