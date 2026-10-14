# PBX3 documentation

Operator and installer guides for **PBX3** — usable as the **lab runbook** while the product hardens (edit freely; treat drafts as living notes).

!!! tip "Lab map"
    Admin SPA `https://app.pbx3.com` · Golden API `https://08jzwn.pbx3.com:44300/api` · Gatekeeper `https://control.pbx3.com` · SBC `https://sbc.pbx3.com/admin` · bucket `08jzwn-pbx3`

## Big picture

```mermaid
flowchart TB
    subgraph phones["Phones and softphones"]
      P1["Handset"]
      P2["Softphone"]
    end

    SBC{{"SBC<br/>SIP edge · single front door"}}

    subgraph fleet["PBX instances"]
      direction LR
      N1["Instance A<br/>Asterisk + API + tenants"]
      N2["Instance B<br/>Asterisk + API + tenants"]
      N3["Instance C<br/>Asterisk + API + tenants"]
    end

    S3[("Org S3 store<br/>catalog · backups · recordings")]
    GK["Gatekeeper<br/>fleet control plane"]
    SPA["Admin SPA<br/>app.pbx3.com · instance + Fleet"]
    CAR[["PSTN / Carriers"]]

    P1 --> SBC
    P2 --> SBC
    SBC -->|"routes by tenant domain"| N1
    SBC --> N2
    SBC --> N3
    SBC <-->|"trunks"| CAR

    N1 -. "backups / meta" .-> S3
    N2 -. "backups / meta" .-> S3
    N3 -. "backups / meta" .-> S3

    GK -->|"catalog truth"| S3
    GK -->|"repoint / project"| SBC
    GK -.->|"fleet APIs"| fleet
    SPA -->|"instance Sanctum"| fleet
    SPA -->|"fleet token"| GK

    classDef edge fill:#f4d,stroke:#333,color:#000;
    class SBC edge;
```

**Calls** go Phones → **SBC** → **instances**. The SBC is a **signaling border / proxy-registrar** (RTP stays on the home — not a media B2BUA).  
**S3** and the **Gatekeeper** are control-plane memory and orchestration — calls keep working if they are down.

**Prefer an AI coding agent?** Optional co-pilot path: [AI-assisted operations](getting-started/ai-assisted.md) / [AI-assisted install](installation/ai-assisted-install.md). Classical CLI pages remain a complete path — start at [What is PBX3?](getting-started/what-is-pbx3.md), [Sign in](getting-started/sign-in.md), or [Install](installation/requirements.md).

Integrators: [Instance API](api/index.md) (Sanctum JSON API on each node `:44300`).
