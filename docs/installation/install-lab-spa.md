# Admin SPA

**Audience:** operators and Lab installers. You do **not** need to build or host the SPA yourself for production or cloud fleets.

## Product path (shared SPA)

Open **[https://app.pbx3.com](https://app.pbx3.com)** — the public GitHub Pages admin for **pbx3-oss**.

| You need | You do **not** need |
|----------|---------------------|
| A reachable node API (`https://{fqdn}:44300/api`) and/or your fleet catalog HTTPS URL | Your own Pages site or `npm run build` |
| Org bucket CORS allowing origin `https://app.pbx3.com` (fleet) | Baking a separate SPA per fleet |
| Node API CORS allowing the same origin (for Bearer login) | Installing **pbx3spa** on the PBX |

**Solo:** open **app.pbx3.com** → enter email, password, and API URL. Leave catalog unset / unused. See [Solo trial](../getting-started/solo-trial.md).

**Fleet:** open **app.pbx3.com** → use the default catalog, or **Switch fleet catalog…** (or `?catalog=`) to your `…/catalog/instance-index.json` → pick an instance → sign in. Details: [Sign in](../getting-started/sign-in.md) · [SPA catalog URL and Pages CORS](../cloud/spa-catalog-cors.md).

Self-hosting a fork of **pbx3spa** is optional (private branding / air-gap only).

---

## Lab LAN path (Vite on your PC)

Lab Garage catalogs are often **private HTTP** on the LAN (`http://192.168.x.x`). The public Pages SPA cannot reach those URLs from the browser. For the [Lab install sequence](install-lab-worksheet.md), run the **Vite dev server** on your PC so proxies keep the browser on `http://localhost:5173`.

**Needs:** Node.js **18+** (20 LTS fine), Git, home LAN IP (example `192.168.1.31`), control LAN IP (example `192.168.1.33`). Home API already up ([Lab home](install-lab-home.md) `/up` → 200).

### Clone and env

```bash
git clone --depth 1 https://github.com/pbx3-oss/pbx3spa.git
cd pbx3spa
```

Create **`.env.development`** (not in git). Edit IPs if yours differ (`.31` = home PBX, `.33` = Gatekeeper/Garage):

```env
VITE_API_PROXY_TARGET=https://192.168.1.31:44300
VITE_DEFAULT_API_BASE_URL=http://localhost:5173/api

VITE_CATALOG_PROXY_TARGET=http://192.168.1.33
VITE_INSTANCE_DIRECTORY_URL=/dev-catalog/catalog/instance-index.json

VITE_FLEET_GATEKEEPER_PROXY_TARGET=http://192.168.1.33
VITE_FLEET_GATEKEEPER_URL=/fleet-gk
```

| Variable | What it does |
|----------|----------------|
| `VITE_API_PROXY_TARGET` | Browser `/api` → home `https://…:44300` (proxy skips TLS verify — Lab snakeoil OK) |
| `VITE_DEFAULT_API_BASE_URL` | Login form pre-fill: `http://localhost:5173/api` |
| `VITE_CATALOG_PROXY_TARGET` + `VITE_INSTANCE_DIRECTORY_URL` | Fleet picker via Garage catalog (no browser CORS to Garage) |
| `VITE_FLEET_GATEKEEPER_*` | **Fleet console** → Gatekeeper on the control host |

### Run Vite

```bash
npm install
npm run dev
```

Open **http://localhost:5173**. Leave the terminal running (Ctrl+C to stop).

Do **not** open `https://192.168.1.31:44300` in the browser (snakeoil). Always use the Vite URL for Lab.

### Sign in (Lab)

**Instance admin** (tenants, extensions, Commit): email/password from the [home installer](install-lab-home.md). API base `http://localhost:5173/api`.

**Fleet console** (catalog, Register instance): [control installer](install-lab-control.md) fleet email/password. Then [adopt the home](install-lab-adopt.md).

Modes use different passwords. Do not mix them.

### WebRTC Line test (optional)

After [Lab SBC §4 WSS](install-lab-sbc.md#4-webrtc--wss-on-the-lab-sbc-required-for-spa-line-test) and a **WebRTC** extension: Extensions → detail → **Line test**. Override WSS to `wss://192.168.1.85:8089/ws` (trust the lab self-signed cert first). SIP domain = tenant FQDN; user = **shortuid**.

## Next (Lab sequence)

[Adopt a Lab home into Fleet](install-lab-adopt.md).
