# Extensions

Routes: `/extensions`, `/extensions/new`, `/extensions/:id`

1. List → open extension (or New).
2. Set name, number, devices / credentials as needed.
3. **Save** → top bar **Commit**.

Until Commit, Asterisk may still serve the previous config.

Phones usually register through the SBC to a **tenant FQDN**, not the raw instance IP — keep Host / SIP domain aligned with the tenant.

Config download (RPS / setting server) is separate from SIP — see [Desk phone RPS enrollment](phone-provisioning-rps.md). The SPA shows the provision URL and **Reset provision state** (Once) on the extension.
