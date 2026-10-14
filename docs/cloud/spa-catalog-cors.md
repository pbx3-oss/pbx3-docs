# SPA catalog URL and Pages CORS

## Production model

- **One shared admin SPA:** **[https://app.pbx3.com](https://app.pbx3.com)** (GitHub Pages for `pbx3-oss/pbx3spa`). Operators and builders do **not** deploy their own SPA.
- Instances remain **API-only** (`:44300`).
- **Default** catalog may be baked in CI (`VITE_INSTANCE_DIRECTORY_URL`). Operators attach **other fleets** with **Switch fleet catalog…** or `?catalog=` → their `https://…/catalog/instance-index.json`.
- Each fleet’s org bucket must CORS-allow origin **`https://app.pbx3.com`**.

## Checklist (every fleet bucket / node)

1. Bucket CORS: include `https://app.pbx3.com` (and `http://localhost:5173` if you use Lab Vite). Methods at least `GET`, `HEAD`.
2. **Each** node API CORS: same SPA origin + `Authorization` (lab nodes often allow `*`).
3. Solo trial: no catalog required — API URL only ([Rule 6 / Solo trial](../getting-started/solo-trial.md)).
4. Lab Vite may proxy `/dev-catalog` to Garage/S3 to bypass browser CORS — **Lab LAN only**; see [Admin SPA](../installation/install-lab-spa.md).

## Example CORS origins

```json
[
  "http://localhost:5173",
  "http://127.0.0.1:5173",
  "https://app.pbx3.com",
  "https://pbx3-oss.github.io"
]
```

## Lab catalog (reference cloud fleet)

`https://08jzwn-pbx3.s3.us-east-1.amazonaws.com/catalog/instance-index.json`
