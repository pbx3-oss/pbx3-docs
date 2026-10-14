# Endpoint reference

Digest of instance routes under `/api`. Paths are relative to the [base URL](index.md#base-url).

**Body accuracy:** Required create fields and many update lists below follow Laravel validation style. Where marked *verify*, confirm against the controller `$updateableColumns` or **`GET /schemas`**. Known path/method mistakes from older docs have been corrected against `routes/api.php`.

**Ability legend:** T = `ability:admin,tenant` · R = `ability:admin,recordings` · A = `abilities:admin` · F = `fleet.token` · S = any Sanctum

---

## Schemas

#### GET /schemas (T)

Returns JSON keyed by resource with `read_only`, `updateable`, and `defaults` for SPA forms. Not OpenAPI.

---

## Agents (T)

#### GET /agents
#### GET /agents/export/pdf
#### GET /agents/{agent}
#### POST /agents

**Body (required):**

```
'pkey' => 'required|integer|min:1000|max:9999',
'cluster' => 'required|exists:cluster,pkey',
'name' => 'required|alpha_dash',
'passwd' => 'required|integer|min:1000|max:9999',
```

#### PUT /agents/{agent}

**Body (updateable — verify):**

```
'cluster' => 'exists:cluster,pkey',
'name' => 'alpha_dash',
'passwd' => 'integer',
'queue1' => 'exists:queue|nullable',
'queue2' => 'exists:queue|nullable',
'queue3' => 'exists:queue|nullable',
'queue4' => 'exists:queue|nullable',
'queue5' => 'exists:queue|nullable',
'queue6' => 'exists:queue|nullable',
```

#### DELETE /agents/{agent}

---

## Extensions (T)

#### GET /extensions
#### GET /extensions/live
#### GET /extensions/export/pdf
#### GET /extensions/{extension}
#### GET /extensions/{extension}/runtime
#### GET /extensions/{extension}/cos

#### POST /extensions

Create shape depends on type (mailbox / provisioned / unprovisioned / webrtc). Minimum typically includes `pkey` and `cluster`. *Verify* create rules in `ExtensionController`.

Example (historical mailbox-style):

```
'pkey' => 'required',
'cluster' => 'required|exists:cluster,pkey',
```

#### PUT /extensions/{extension}

**Body (updateable — verify):**

```
'active' => 'in:YES,NO',
'callbackto' => 'in:desk,cell',
'callerid' => 'integer|nullable',
'cellphone' => 'integer|nullable',
'celltwin' => 'in:ON,OFF',
'cluster' => 'exists:cluster,pkey',
'devicerec' => 'in:None,OTR,OTRR,Inbound,Outbound,Both',
'dvrvmail' => 'exists:ipphone,pkey|nullable',
'location' => 'in:local,remote',
'protocol' => 'in:IPV4,IPV6',
'provision' => 'string|nullable',
'provisionwith' => 'in:IP,FQDN',
'sndcreds' => 'in:No,Once,Always',
'transport' => 'in:udp,tcp,tls,wss',
'vmailfwd' => 'email|nullable',
```

#### PUT /extensions/{extension}/runtime
#### PUT /extensions/{extension}/cos
#### POST /extensions/{extension}/regenerate-sip-password
#### DELETE /extensions/{extension}

---

## Conferences (T)

#### GET /conferences · GET /conferences/export/pdf · GET /conferences/{conference}
#### POST /conferences · PUT /conferences/{conference} · DELETE /conferences/{conference}

Bodies: *verify* controller / schemas.

---

## Queues (T)

#### GET /queues
#### GET /queues/export/pdf
#### GET /queues/{queue}
#### POST /queues

**Body (required — verify):**

```
'pkey' => 'required',
'cluster' => 'required|exists:cluster,pkey',
```

#### PUT /queues/{queue}

**Body (updateable — verify):**

```
'conf' => 'string',
'cluster' => 'exists:cluster,pkey',
'devicerec' => 'in:None,OTR,OTRR,Inbound',
'greetnum' => 'regex:/^usergreeting\\d{4}$/',
'options' => 'alpha',
```

#### DELETE /queues/{queue}

---

## IVRs (T)

#### GET /ivrs · GET /ivrs/export/pdf · GET /ivrs/{ivr}
#### POST /ivrs

**Body (required):**

```
'pkey' => 'required',
'cluster' => 'required|exists:cluster,pkey',
```

#### PUT /ivrs/{ivr}

Option/alert/tag fields `0`–`11`, plus `description`, `cluster`, `greetnum`, `listenforext`, `timeout`, etc. *Verify* full list in controller.

#### DELETE /ivrs/{ivr}

---

## Inbound routes (T)

#### GET /inboundroutes · GET /inboundroutes/export/pdf · GET /inboundroutes/{inboundroute}
#### POST /inboundroutes

**Body (required — verify):**

```
'pkey' => 'required',
'cluster' => 'required|exists:cluster,pkey',
'trunkname' => 'required',
```

#### PUT /inboundroutes/{inboundroute}

**Body (updateable — verify):**

```
'active' => 'in:YES,NO',
'alertinfo' => 'string',
'closeroute' => 'string',
'cluster' => 'exists:cluster,pkey',
'description' => 'string',
'disa' => 'in:DISA,CALLBACK|nullable',
'disapass' => 'alpha_num|nullable',
'inprefix' => 'integer|nullable',
'moh' => 'in:ON,OFF',
'openroute' => 'string',
'swoclip' => 'in:YES,NO',
'tag' => 'alpha_num|nullable',
```

(Also schedule-mode / route-profile fields for day-parts — *verify*.)

#### DELETE /inboundroutes/{inboundroute}

---

## Day timers (T)

#### GET /daytimers
#### GET /daytimers/{daytimer}
#### POST /daytimers

**Body (verify — includes DOW ranges / schedule mode):**

```
'cluster' => 'required|exists:cluster,pkey',
'datemonth' => 'in:*,1,2,...31',
'dayofweek' => 'string',   # * | mon | tue | … | ranges e.g. mon-fri
'desc' => 'string',
'month' => 'in:*,jan,feb,...dec',
'timespan' => 'regex:…',   # half-open local times; see admin day-timers guide
```

#### PUT /daytimers/{daytimer}
#### DELETE /daytimers/{daytimer}

---

## Holiday timers (T)

#### GET /holidaytimers · GET /holidaytimers/{holidaytimer}
#### POST /holidaytimers · PUT /holidaytimers/{holidaytimer} · DELETE /holidaytimers/{holidaytimer}

Times historically stored as epoch seconds. Bodies *verify*.

---

## Route profiles (T)

Named open/closed/(mode) destination profiles used by day-parts / inbound schedule mode.

#### GET /routeprofiles
#### GET /routeprofiles/{routeprofile}
#### POST /routeprofiles
#### PUT /routeprofiles/{routeprofile}
#### DELETE /routeprofiles/{routeprofile}

Bodies: *verify* `RouteProfileController`.

---

## Class of service (T)

**Rules** = deny packs; **profiles** = Standard + After-hours rule lists assigned to extensions. Prefer profiles for assignment; junction resources remain for lab dual-read until merge confidence.

| Resource | Role |
|----------|------|
| **cosrules** | Deny-pattern CRUD; `orideopen` / `orideclosed` = **Tenant-wide** |
| **cosprofiles** | Profile CRUD; `open_rules` / `closed_rules`; `is_default` **read-only for operators** (fixed Default — bootstrap/convert/seed only) |

Extension field **`cos_profile`** (profile `pkey`) on extension show/update.

### cosprofiles

#### GET /cosprofiles · GET /cosprofiles/{cosprofile}
#### POST /cosprofiles

**Body (verify):** `cluster`, `cname`, `description`, `active`, `open_rules[]`, `closed_rules[]`.  
`is_default` is not client-elected: first profile on an empty tenant becomes Default; later creates are non-default.

#### PUT /cosprofiles/{cosprofile}

Cannot change `is_default` (422). Edit rule lists on the Default profile to change default dial policy.

#### DELETE /cosprofiles/{cosprofile}

Cannot delete the Default profile (409).

Delete blocked if profile is default or still assigned to extensions.

### coscloses

#### GET /coscloses · GET /coscloses/{cosclose}
#### POST /coscloses

**Body (verify):**

```
'ipphone_pkey' => 'exists:ipphone,pkey',
'cos_pkey' => 'exists:cos,pkey',
```

#### PUT /coscloses/{cosclose}
#### DELETE /coscloses/{cosclose}

### cosopens

#### GET /cosopens · GET /cosopens/{cosopen}
#### POST /cosopens
#### PUT /cosopens/{cosopen}
#### DELETE /cosopens/{cosopen}

### cosrules

#### GET /cosrules · GET /cosrules/{classofservice}
#### POST /cosrules

**Body (verify):**

```
'pkey' => 'required|alpha_dash',
'dialplan' => 'required',
```

Tenant-wide: `orideopen` / `orideclosed` (`YES`/`NO`). Rule `defaultopen` / `defaultclosed` demoted (seed/convert only).

#### PUT /cosrules/{classofservice}
#### DELETE /cosrules/{classofservice}

---

## Greetings (T)

#### GET /greetings
#### GET /greetings/{greeting} — download
#### POST /greetings — multipart

```
'greeting' => 'required|file|mimes:wav,mpeg',
```

#### DELETE /greetings/{greeting}

---

## Greeting records (T)

Metadata + replace for greeting files.

#### GET /greetingrecords
#### GET /greetingrecords/{greetingrecord}
#### GET /greetingrecords/{greetingrecord}/download
#### POST /greetingrecords
#### PUT /greetingrecords/{greetingrecord}
#### POST /greetingrecords/{greetingrecord}/replace
#### DELETE /greetingrecords/{greetingrecord}

Bodies: *verify* `GreetingRecordController`.

---

## Destinations (T)

#### GET /destinations

List of currently valid destinations in the database (read-only).

---

## CDR and Home pulse (T)

#### GET /cdr
#### GET /home/pulse

Home dashboard / pulse payload. Query params *verify* controllers.

---

## Help messages

#### GET /helpcore · GET /helpcore/{helpcore} (T)
#### POST /helpcore · PUT /helpcore/{helpcore} · DELETE /helpcore/{helpcore} (A)

---

## System commands

### Tenant-accessible (T)

#### GET /syscommands/commitstatus
#### GET /syscommands/commit
#### GET /syscommands/pbxrunstate
#### GET /syscommands/sysnotes

### Admin (A)

#### GET /syscommands — index / help list
#### GET /syscommands/reboot
#### GET /syscommands/start
#### GET /syscommands/stop
#### PUT /syscommands/hostname
#### PUT /syscommands/dns
#### PUT /syscommands/smtp
#### GET /syscommands/timezones
#### PUT /syscommands/timezone
#### PUT /syscommands/icmp

---

## System globals

#### GET /sysglobals (T) — read instance globals
#### PUT /sysglobals (A) — update

Updateable keys include site/network/recording/SIP defaults (e.g. `FQDN`, `COUNTRYCODE`, `default_outbound_dialplan`, …). Full list: *verify* `SysglobalController` / schemas. Historical UPPERCASE keys still appear in many deployments.

---

## Recordings (R)

#### GET /recordings
#### GET /recordings/{recording}/stream
#### GET /recordings/{recording}/download

---

## Custom apps (A)

Tenant table `appl`. API sets `id` / `shortuid` on create — do not send them.

#### GET /customapps · GET /customapps/{customapp}
#### POST /customapps

**Body:**

```
'pkey' => 'required',           # app name
'cluster' => 'required|exists:cluster,pkey',
'cname' => 'string|nullable',
'description' => 'string|nullable',
'span' => 'in:Internal,External,Both,Neither',
'active' => 'in:YES,NO',
'striptags' => 'in:YES,NO',
'directdial' => 'integer|nullable',
'extcode' => 'string|nullable',
```

#### PUT /customapps/{customapp} · DELETE /customapps/{customapp}

---

## Devices (removed 2026-08-25)

Provisioning templates (`/devices`) and the instance **Device** table were removed — no per-SKU Device matrix (#28). Home phone provision (Phase A) is a separate listener on **:41363** (`#INCLUDE` vendor streams on `ipphone.provision`) — see pbx3 **`PROVISIONING_SERVER_REQUIREMENTS.md`**. Extension type lives on **`ipphone.device`** (General SIP / WebRTC / MAILBOX). Existing DBs: `sqlite_device_drop.sql`.

---

## Backups (A)

#### GET /backups
#### GET /backups/new — take a new backup
#### GET /backups/{backup} — download
#### GET /backups/archive/{backup_stamp}/download-url — stamp `\d{8}T\d{6}Z`
#### POST /backups — upload zip (multipart)
#### POST /backups/restore-from-archive
#### PUT /backups/{backup} — restore selected elements

**Restore body (verify):**

```
'restoredb' => 'boolean',
'restoreasterisk' => 'boolean',
'restoreusergreeting' => 'boolean',
'restorevmail' => 'boolean',
'restoreldap' => 'boolean',
```

#### DELETE /backups/{backup}

---

## Snapshots (A)

DB snapshot set (zip).

#### GET /snapshots · GET /snapshots/new · GET /snapshots/{snapshot}
#### POST /snapshots · PUT /snapshots/{snapshot} · DELETE /snapshots/{snapshot}

---

## Outbound routes (A)

#### GET /routes · GET /routes/export/pdf · GET /routes/{route}
#### POST /routes

**Body (required — verify):**

```
'pkey' => 'required',
'cluster' => 'required|exists:cluster,pkey',
```

#### PUT /routes/{route}

**Body (updateable — verify):**

```
'active' => 'in:YES,NO',
'auth' => 'in:YES,NO',
'cluster' => 'exists:cluster,pkey',
'desc' => 'alpha_dash',
'dialplan' => 'string',
'path1' => 'nullable',
'path2' => 'nullable',
'path3' => 'nullable',
'path4' => 'nullable',
'strategy' => 'in:hunt,balance',
```

#### DELETE /routes/{route}

---

## Dial aliases / dial prefixes (A)

Cross-tenant short dial rows on this instance. **Instance admin only** (not `tenant` ability).

!!! note "Fleet product path"
    Production sister-site short dial is configured in **Fleet → Site Groups**, not by inventing meshes here. See [Site Groups](../fleet/site-groups.md). When the dial-cohort feature is on, Sanctum forbids create/update/delete of cross-tenant prefixes; managed (`source=cohort`) rows are read-only.

#### GET /dialaliases · GET /dialaliases/{dialalias}
#### POST /dialaliases · PUT /dialaliases/{dialalias} · DELETE /dialaliases/{dialalias}

Bodies: *verify* `DialAliasController`.

---

## Certificates (A)

#### GET /certificates/active
#### GET /certificates/letsencrypt
#### POST /certificates/letsencrypt/setup
#### POST /certificates/letsencrypt/sync
#### POST /certificates/letsencrypt/renew
#### GET /certificates/custom
#### POST /certificates/custom
#### DELETE /certificates/custom

Bodies: *verify* `CertificateController`. See also TLS admin guides.

---

## Firewall (A)

Declarative **UFW** allow-list (not Shorewall). `GET` returns allow rows; `POST` validates/saves; `PUT` applies / restarts UFW via the home apply script.

#### GET /firewalls · POST /firewalls · PUT /firewalls
#### GET /firewalls/ipv4 · POST /firewalls/ipv4 · PUT /firewalls/ipv4
#### GET /firewalls/ipv6 · POST /firewalls/ipv6 · PUT /firewalls/ipv6

**POST body:** allow-list payload (*verify* `FirewallController`). IPv4 aliases remain for older clients.
---

## Asterisk config files (A)

#### GET /astfiles
#### GET /astfiles/{filename}
#### PUT /astfiles/{filename}

---

## Logs (A)

#### GET /logs
#### GET /logs/retention · PUT /logs/retention
#### GET /logs/archive · GET /logs/archive/download-url
#### GET /logs/cdrs{limit} — e.g. `/logs/cdrs50` (limit is a path suffix, **no** slash before the number)
#### GET /logs/{logfile} · GET /logs/{logfile}/download

---

## Tenants (A)

Internally “clusters”. On **fleet** nodes, prefer [fleet create/delete](#fleet-service-token) over Sanctum POST/DELETE when lifecycle is locked.

#### GET /tenants · GET /tenants/export/pdf · GET /tenants/{tenant}
#### POST /tenants

**Body (required — verify):**

```
'pkey' => 'required|string',
'description' => 'required|string',
```

On create, outbound **MainOut** may be seeded from `globals.default_outbound_dialplan` when that feature is enabled.

#### PUT /tenants/{tenant}

Many tenant settings (`masteroclo`, ring delays, recording limits, LDAP, …). *Verify* `TenantController` / schemas.

#### DELETE /tenants/{tenant}

Deletes the tenant and **all** of its dependent rows (Sanctum path). Fleet wipe uses `DELETE /fleet/tenants/{tenant}` instead.

---

## Trunks (A)

#### GET /trunks · GET /trunks/live · GET /trunks/export/pdf · GET /trunks/{trunk}
#### POST /trunks

**Body (required — verify):**

```
'pkey' => 'required',
'technology' => 'required|in:SIP,IAX2',
'cluster' => 'required|exists:cluster,pkey',
'username' => 'required',
'host' => 'required',
```

#### PUT /trunks/{trunk} · DELETE /trunks/{trunk}

Updateable fields include `active`, `callerid`, `host`, `password`, `transform`, recording flags, etc. *Verify* controller.

---

## Asterisk AMI (A)

Blocking AMI requests. Function names are **case-sensitive** to match Asterisk docs. Response field sets are not fully documented — inspect with a REST client.

#### GET /astamis — list available actions (as implemented)
#### GET /astamis/CoreSettings · GET /astamis/CoreStatus
#### GET /astamis/ExtensionState/{id}{context?}
#### GET /astamis/MailboxCount/{id} · GET /astamis/MailboxStatus/{id}
#### GET /astamis/QueueStatus/{id} · GET /astamis/QueueSummary/{id}
#### GET /astamis/Reload
#### GET /astamis/{action}/{id?} — generic list/instance actions (SIPpeers, CoreShowChannels, …)

#### POST /astamis/originate

**Body (verify):**

```
'target' => 'required|integer',
'caller' => 'required|numeric',
'context' => 'required|alpha_dash',
'clid' => 'nullable|numeric',
```

### AstDB

#### GET /astamis/DBget/{id}/{key}
#### PUT /astamis/DBput/{id}/{key}/{value}
#### DELETE /astamis/DBdel/{id}/{key}

### Soft hangup

#### DELETE /astamis/Hangup/{id}/{key}

Example channel `PJSIP/44107-00000155`:

```
DELETE /api/astamis/Hangup/PJSIP/44107-00000155
```

---

## Fleet posture (S)

#### GET /fleet-posture

Sanctum (any authenticated user). Returns whether this instance is fleet-joined / related posture for the SPA.

---

## Fleet (service token)

Auth: **`Authorization: Bearer <PBX3_FLEET_SERVICE_TOKEN>`** — not Sanctum. Used by Gatekeeper / ops for mobility and fleet tenant lifecycle on the **node**.

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/fleet/preflight` | Preflight checks |
| GET | `/fleet/egress-qualify` | Egress qualify |
| POST | `/fleet/tenants` | Create tenant (fleet path) |
| POST | `/fleet/tenants/{tenant}/export` | Export tenant blob |
| POST | `/fleet/tenants/import` | Import tenant |
| POST | `/fleet/commit` | Commit on node |
| POST | `/fleet/certificates/sync` | Cert sync |
| DELETE | `/fleet/tenants/{tenant}` | Wipe tenant data on node |

Bodies and job semantics: see fleet ops docs and `pbx3api` working notes (`FLEET_TENANT_CREATE.md`). Product **Fleet Delete** (catalog + SBC domain + durable job) is a separate control-plane track.

**Not covered here:** Gatekeeper `/api/v1/…` fleet UI auth and catalog mutate APIs.

---

## Smoke test (A)

#### GET /test/admin-only

Returns `{"message":"Admin access granted"}` when the token has `admin`.

---

## Fallback

Unknown paths under `/api` return JSON `404` with `Unauthorised/Page Not Found`.
