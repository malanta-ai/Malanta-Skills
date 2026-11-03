---
name: malanta-profile-api-public
description: >
  BETA Malanta Profile API (Gold Query). Advanced skill for indicator capabilities
  and attribution: services, tags, web (JS functions/capabilities), TLS/JARM,
  redirections, and intelligence attribution on a domain, FQDN, or IP. Use after
  basic triage when the user needs what the indicator does and who it ties to -
  not for first-pass IoPA/IoC verdicts (use malanta-api-public for reputation,
  clusters, batch, history). Work in progress; feedback welcome.
version: 1.0-public-beta
provider: Malanta
category: Pre-Attack Prevention
status: beta
---

# Malanta Profile API Skill (BETA)

**Status:** BETA - work in progress. Schemas, coverage, and workflows may change. Feedback welcome: support@malanta.ai  
**Version:** 1.0-public-beta  
**API name:** Malanta Profile API (Gold Query)  
**Profile API docs version:** 1.0 (October 2026)  
**Tier:** Advanced (not first-pass triage)  
**Compatibility:** Any AI agent or model that can call HTTPS APIs and follow structured instructions  
**Canonical docs:** https://help.malanta.ai/en/article/profile-api-integration-guide-1akmgbq/

Use this skill for **advanced** Observable Profile calls: show what an indicator **can do** (capabilities) and **who/what it is attributed to** - not to replace reputation or cluster triage.

**Unique value (lead with these):** services, tags, web (page + JS functions/capabilities), TLS/JARM, redirections (`redirects_to`), intelligence attribution.

**Companion skill (default / basic path):** For IoPA/IoC reputation, attack clusters, batch triage, WHOIS/DNS/cert history, and standard CHAINED investigations, load `malanta-api-public` first. Profile does **not** replace `/reputation` or `/clusters`. Escalate to Profile when the analyst needs capability depth or attribution detail.

When presenting results, state that Profile is **BETA** if the user may integrate or automate against it.

---

## 0. Security and Safe Use

### API credentials

- Never print, log, echo, commit, or include API keys in responses or artifacts
- Store the key in the environment or a secrets manager (for example `MALANTA_API_KEY`)
- Auth header: `x-api-key`. Do not put the key in URLs or query strings
- If a key is exposed, rotate it

### Response integrity

- Never fabricate Profile sections, tags, JARM fingerprints, APT names, or related observables
- Empty/`{}`/`[]` means no data for that section - say so
- Separate API fields from analytical judgment; use High / Medium / Low confidence on judgments
- HTTP 200 with empty sections is success, not evidence the indicator is safe or malicious
- `dangerous: true` on JS capabilities marks a security-sensitive capability - **not** a malicious verdict

### Handling enrichment data

- Prefer summarizing WHOIS contacts; avoid unnecessary exposure of personal contact details
- Do not pivot on privacy-proxy or redacted WHOIS contacts
- Do not treat large `shared_hosting` sets on CDN/cloud IPs as strong operator linkage

---

## 1. What Profile Is (advanced capability + attribution)

Malanta is a Pre-Attack Prevention platform. Profile is an **advanced** Gold Query surface: current-state view of an indicator's **capabilities** and **attribution**, after basic triage (or when the user explicitly asks for Profile depth).

**Key terminology:** Adversary Infrastructure Identity, common identity attributes, IoPAs.  
**Category:** Pre-Attack Prevention.  
**Positioning anchors:** MITRE ATT&CK **TA0042 (Resource Development)** and **TA0043 (Reconnaissance)**.

### Unique value vs other Malanta APIs

Other Malanta routes answer **is this attacker infrastructure?** and **what else belongs to the operator?** Profile answers **what can this indicator do right now?** and **what attribution signals attach to it?**

| Profile unique (lead) | What it shows | Other APIs do not center this |
|----------------------|---------------|-------------------------------|
| **Services** | Open ports, schemes, banners, probe success/failure, endpoint certs | `/reputation`, `/clusters`, WHOIS/DNS history |
| **Tags** | Infrastructure / classification tags (`key`, `name`, `family`) | Not a substitute for IoPA/IOC labels |
| **Web** | Page title/body hashes, favicon hashes, CSS/JS resources | Specialized DNS/cert routes |
| **Functions / JS capabilities** | Per-script and unattributed capabilities (`id`, `dangerous`), notable functions | Nowhere else on Malanta public routes |
| **TLS** | JARM fingerprint + negotiated TLS (version, cipher, ALPN, SNI) | Not on `/reputation` |
| **Redirections** | `related_observables` with `related_by: redirects_to` (+ method/persistence/status) | Not the focus of `/info` or `/reputation` |
| **Attribution** | `intelligence.attribution.apts`, categorized `attributes[]` | Complements APT on reputation; Profile is the deep attribute view |

Supporting context on the same response (do not lead with these unless asked): `state`, `registration`, `network`, `dns`, `resolves_to`, `shared_hosting`. Those overlap specialized WHOIS / DNS / ip-ranges routes - Profile's differentiator remains capabilities + attribution.

### Investigation tiering

| Tier | Skill / routes | Job |
|------|----------------|-----|
| **Basic (default)** | `malanta-api-public`: `/info`, `/reputation`, `/clusters`, batch | Verdict, IoPA/IOC, score, campaign membership |
| **Enrichment** | Companion: WHOIS / DNS / certificates / ip-ranges (history + pagination) | Registration and infra history depth |
| **Advanced** | **This skill:** `/profile` | Capabilities (services, tags, web, functions, TLS, redirections) + attribution |

Do **not** call Profile as the first step for "is it malicious?", batch triage, or watchlist monitoring. Escalate to Profile when the user asks for capabilities, live services, TLS/JARM, web/JS behavior, redirects, tags, or attribution detail - or after a MALICIOUS hit when deeper capability context is needed.

### When to use Profile vs other routes

| Need | Use |
|------|-----|
| Capabilities / what does it expose / services / TLS / web / JS functions / redirects / tags / attribution | **`/profile` (this skill, advanced)** |
| Verdict, IoPA/IOC labels, score band, cluster membership | `/reputation` + `/clusters` (companion, basic) |
| Full WHOIS / DNS / cert history with pagination | Specialized history routes (companion) |
| Batch classify many indicators | `POST .../reputation` (companion) |
| "Quick check" / first pass | Companion `/info` or `/reputation` - not Profile |

### Constraints

| Rule | Detail |
|------|--------|
| Current-state only | No `history` / `since` / `until` on `/profile` |
| Domain / FQDN / IP | Three GET routes; no batch POST |
| BETA | Treat as WIP; do not hard-depend on every nested field in production without validation |
| Not a verdict API | Empty Profile ≠ safe; tags/capabilities ≠ IoPA |

---

## 2. Malanta API Configuration

```
Base URL:  https://app.malanta.ai/data
Version:   /v1
Auth:      x-api-key header (tenant key, typically malanta_ prefix)
Methods:   GET only for Profile (no batch POST)
Response:  { data, query, meta }  - no pagination field
```

### Authentication example

```bash
BASE_URL="https://app.malanta.ai/data"
API_KEY="${MALANTA_API_KEY}"

curl -sS "$BASE_URL/v1/domains/example.com/profile" \
  -H "x-api-key: $API_KEY"

curl -sS "$BASE_URL/v1/fqdns/www.example.com/profile" \
  -H "x-api-key: $API_KEY"

curl -sS "$BASE_URL/v1/ips/203.0.113.45/profile" \
  -H "x-api-key: $API_KEY"
```

### Response envelope

```json
{
  "data": {
    "profile": {},
    "related_observables": [],
    "intelligence": {}
  },
  "query": {
    "indicator": {
      "type": "domain-name",
      "value": "example.com"
    }
  },
  "meta": {
    "request_id": "uuid",
    "schema_version": "2.0.0",
    "endpoint": "/v1/domains/{domain}/profile",
    "elapsed_ms": 412,
    "generated_at": "2026-10-14T09:00:00Z",
    "coverage": {
      "first_observed_at": "2026-08-26T08:32:43Z",
      "last_observed_at": "2026-08-26T08:32:43Z"
    }
  }
}
```

- No `pagination` on Profile (single-object responses)
- `meta.request_id` helps Malanta support diagnose failed calls
- `meta.coverage` may include observation span for the entity
- Timestamps are ISO-8601 UTC with trailing `Z`

### Indicator object

| Field | Notes |
|-------|-------|
| `type` | `domain-name`, `fqdn`, `ipv4-addr`, or `ipv6-addr` |
| `value` | Canonical display value |
| `indicator_id` | Stable UUID when resolvable |
| `ip_addr_hex` | Set for IP indicators (32-char IPv4-mapped hex when present) |

### Input validation (pre-call)

| Check | Rule | On failure |
|-------|------|------------|
| `/v1/domains/.../profile` | Must be eTLD+1 (registrable apex) | Use `/v1/fqdns/.../profile` for subdomains |
| `/v1/fqdns/.../profile` | Any well-formed host | - |
| `/v1/ips/.../profile` | IPv4 or IPv6 address string | Do not pass domains or FQDNs |
| Temporal params | `since` / `until` / `history` | Omit - they return **400** on `/profile` |
| Batch | No Profile batch route | Call once per indicator; use companion skill for batch reputation |

---

## 3. Agent Behavior

### Orchestration modes

| Mode | When | Behavior |
|------|------|----------|
| **ATOMIC** (default) | Normal Profile prompts | One best `/profile` route. Suggest next steps. Do not auto-chain |
| **CHAINED** | User says "full profile deep dive", "profile then classify", "profile + IPs", or "expand from profile" | Run a Profile workflow sequence. Present progress. Respect caps |

### Core rules

1. Treat Profile as **advanced**. If the user only asked "is it malicious?" / triage / batch / watchlist, use the companion skill first (or hand off). Do not open with Profile.
2. Route by indicator type: eTLD+1 → domains profile; subdomain/host → fqdns profile; IP → ips profile
3. **Lead with unique value:** capabilities (services, tags, web, JS functions/capabilities, TLS, redirections) then attribution (`intelligence`). Supporting sections (state, registration, network, dns) are secondary unless asked.
4. Profile is **not** a Malanta reputation verdict - never present tags or `dangerous` JS flags as IoPA/IOC
5. Apply noise filters before pivoting on `shared_hosting` or co-tenant signals
6. Never invent `/ipinfo` or temporal query params on `/profile`
7. Mention **BETA** when integrating or when fields look incomplete; invite feedback to support@malanta.ai when useful
8. Use the **Malanta Profile API** routes defined here only for Profile work

### Result presentation (capabilities first, then attribution)

Structure every Profile answer in this order unless the user asked for a specific section:

1. **Capabilities**
   - **Services:** open ports / schemes / products / banners; note probe `error`s
   - **Tags:** `key` / `name` / `family`
   - **TLS:** JARM + negotiated version / cipher / ALPN / SNI
   - **Web:** title, final_url, favicon hashes; CSS/JS aggregate signals
   - **Functions / JS capabilities:** list notable `capabilities` and `functions`; mark `dangerous: true` as security-sensitive capability (not malware)
   - **Redirections:** `related_observables` where `related_by` is `redirects_to` (method, persistence, status)
2. **Attribution**
   - `intelligence.attribution.apts[]` and `attributes[]` (category, value, apts, countries)
3. **Supporting context** (brief): state/registration (domain/FQDN), network/DNS, `resolves_to` / `shared_hosting`
4. **BETA note** (one line) when shipping to customers or automations
5. If unique sections are empty: say so plainly - "No capability/attribution signal in Profile for this indicator."

**IP Profile:** same capability → attribution order. Note IP profiles **omit** `state`, `registration`, and `dns`.

### Suggest next steps (ATOMIC only - do not auto-run)

| Current result | Priority | Secondary |
|----------------|----------|-----------|
| Strong capability signal (services/tags/web/TLS/redirects) | Corroborate with `/reputation`? | Clusters for campaign map? |
| Non-empty attribution (`intelligence`) | Reputation for IoPA/IOC corroboration | Cluster deep dive |
| Tags suggest loader / family | Reputation + clusters | IP Profile on resolved hosts |
| Interesting redirects | Profile redirect targets (FQDN/URL host) | Reputation on landing host |
| Resolved IPs with rich services | IP Profile on top IPs | IP reputation (companion) |
| Large `shared_hosting` on CDN/cloud | Do not mass-pivot | Solo-tenant IP Profile only |
| Empty capability + attribution | Fall back to companion enrichment | Re-check later |
| User asked "is it malicious?" first | Companion `/reputation` first | Offer Profile as **advanced** capability follow-up |

---

## 4. Prompt-to-Endpoint Map (ATOMIC)

### When this skill should activate (advanced)

Activate Profile when the user asks for capabilities, live behavior, or attribution depth - not for first-pass classification.

| User intent (advanced) | Endpoint / section |
|------------------------|--------------------|
| What can this indicator do / capabilities / live behavior | Matching `/profile` - lead services, tags, web, functions, TLS, redirects |
| Open services / ports / banners | `/profile` → `profile.services` |
| Tags / infra classification tags | `/profile` → `profile.tags` |
| Web page / favicon / JS capabilities / functions | `/profile` → `profile.web` |
| TLS / JARM fingerprint | `/profile` → `profile.tls` |
| Redirect chain / where does it send users | `/profile` → `related_observables` (`redirects_to`) |
| Attribution / APT attributes on the observable | `/profile` → `intelligence` |
| Full Profile (apex) | `GET /v1/domains/{domain}/profile` |
| Full Profile (subdomain / host) | `GET /v1/fqdns/{fqdn}/profile` |
| Full Profile (IP) | `GET /v1/ips/{ip}/profile` |

### Hand off to companion skill (basic / other APIs)

| User intent | Call via companion skill |
|-------------|--------------------------|
| Is it malicious / IoPA / IOC / score / quick check | `GET /v1/domains\|ips/{x}/reputation` or `/info` |
| Attack cluster / campaign | `GET /v1/clusters?indicator=` or by id |
| Full WHOIS / DNS / cert history with pagination | Specialized history routes |
| Batch triage / watchlist | `POST .../reputation` |

### Out of scope for this skill

First-pass triage and batch classification (use companion). Malware sandboxing, file hashes, CVE lookup, email header parsing, PCAP decode, URL screenshots, dark web monitoring, batch Profile, temporal Profile queries, inventing non-Profile routes beyond documented handoffs.

---

## 5. Execution Reference

### 5.1 GET /v1/domains/{domain}/profile

Consolidated current-state enrichment for a **registrable domain** (eTLD+1).

```bash
curl -sS "$BASE_URL/v1/domains/example.com/profile" \
  -H "x-api-key: $API_KEY"
```

| `data` section | Role | Contents |
|----------------|------|----------|
| `profile.services.endpoints[]` | **Primary (capabilities)** | Port probes (scheme, service, response, content, certificate, error) |
| `profile.tags[]` | **Primary (capabilities)** | Infrastructure / classification tags (`key`, `name`, `family`) |
| `profile.web` | **Primary (capabilities)** | Page, favicon hashes, CSS / JS resources, functions, capabilities |
| `profile.tls` | **Primary (capabilities)** | JARM fingerprint + negotiated TLS parameters |
| `related_observables[]` → `redirects_to` | **Primary (capabilities)** | Redirect destinations (+ method / persistence / status) |
| `intelligence` | **Primary (attribution)** | APT attribution + categorized attributes (may be `{}`) |
| `related_observables[]` → `resolves_to` / `shared_hosting` | Supporting | Resolution and co-host edges |
| `profile.state` | Supporting | Lifecycle: status, registration_state, takedown |
| `profile.registration` | Supporting | Current WHOIS + significant changes timeline |
| `profile.network` | Supporting | Resolved IPs + ASN / provider / range / PTR |
| `profile.dns` | Supporting | A / MX / NS / TXT |

**Parameters:** none. `since` / `until` / `history` → **400**.  
**Batch:** none.  
**Empty / unknown:** HTTP 200; missing sections as `{}` or `[]`.

### 5.2 GET /v1/fqdns/{fqdn}/profile

Same response **shape** as domain Profile. Use for a specific host (apex or subdomain).

```bash
curl -sS "$BASE_URL/v1/fqdns/www.example.com/profile" \
  -H "x-api-key: $API_KEY"
```

| Scope | Rule |
|-------|------|
| network / DNS / services / TLS / web / tags / related / intelligence | Exact `{fqdn}` |
| `profile.state` + `profile.registration` | Registrable **apex** |
| `query.indicator.type` | `fqdn` |

Subdomain on `/v1/domains/.../profile` → **422**. Always use this route for non-apex hosts.

### 5.3 GET /v1/ips/{ip}/profile

Observable Profile for an IPv4 or IPv6 address.

```bash
curl -sS "$BASE_URL/v1/ips/203.0.113.45/profile" \
  -H "x-api-key: $API_KEY"
```

| Present | Omitted (domain-only) |
|---------|------------------------|
| `tags`, `network` (flat ASN/provider/range/PTR), `services`, `tls`, `web`, `related_observables`, `intelligence` | `state`, `registration`, `dns` |

**Network shape note:** IP uses flat `profile.network` (no `ip_addresses` / `networks[]` wrapper). Domain/FQDN uses the wrapped shape.

**`shared_hosting` note:** On IP Profiles, groups list co-hosted domains/hosts and do **not** include `via` (queried IP is already the host). On domain/FQDN Profiles, `shared_hosting` includes `via` pointing at the shared IP.

### 5.4 Field highlights (agent-facing)

**`profile.state` (domain/FQDN)**  
`domain_status` (`active` / `inactive`), `registration_state` (`unregistered` / `expired` / `on_hold` / `registered`), `ownership_transition` (`sinkholed` / `parked` / null), `takedown.on_hold`.

**`registration.changes`**  
Always included (no `?history` flag). Use `summary` for takedown/event counts; `significant[]` for timeline events (`primary_change`, `changes[]`, snapshot `registration_state`).

**`services.endpoints[]`**  
Fixed schema for success and failure. Failed probes set `error` string; success keeps `error: null`. Certificate on HTTPS endpoints may include fingerprints under `certificate.intelligences`.

**`web.resources.javascript`**  
`obfuscation` is a 0-1 style score when present. `capabilities[].dangerous` / `unattributed_capabilities[].dangerous` flag sensitive capabilities only.

**`intelligence`**  
`attribution.apts[]`, `attributes[]` with `category`, `value`, `apts`, `countries`, optional `legitimacy_status`.

Full field tables: https://help.malanta.ai/en/article/profile-api-integration-guide-1akmgbq/

### 5.5 Errors

| Status | Action |
|--------|--------|
| 200 | Success; empty sections are not an error |
| 400 | Remove temporal params (`history` / `since` / `until`) |
| 401/403 | Auth failure; verify key; include `request_id` for support |
| 422 | Bad input (subdomain on `/domains`, malformed IP) - retry correct route |
| 429 | Back off exponentially |
| 500 / 504 | Retry briefly; then skip step in a chain and note gap |

---

## 6. Noise Filtering

- Skip major provider apexes for expansion (google.com, cloudflare.com, amazonaws.com) unless specifically targeted
- Shared-platform apexes (vercel.app, pages.dev, github.io, workers.dev, netlify.app, icp0.io, URL shorteners): Profile the exact FQDN if asked; do **not** treat platform apex as operator pivot
- Large `shared_hosting` on CDN / cloud / shared hosting: weak co-tenant signal; prefer solo-tenant IPs
- Never pivot on privacy-proxy WHOIS (WhoisGuard, Withheld for Privacy, Domains By Proxy, and similar)
- Skip freemail apex pivots (gmail.com, outlook.com, yahoo.com, and similar)
- JS `dangerous` capabilities are not malware labels
- Profile tags can fire when reputation is IOC-only or clusters are empty - report both; do not invent IoPA from tags alone
- Dedupe resolved IPs before chaining IP Profiles

---

## 7. Concepts Quick Reference

| Term | Meaning |
|------|---------|
| Observable Profile | Advanced current-state view of indicator **capabilities** + **attribution** |
| Capabilities (Profile unique) | Services, tags, web, JS functions/capabilities, TLS/JARM, redirections |
| Attribution (Profile) | `intelligence.attribution` + categorized attributes on the observable |
| BETA | Public WIP; feedback welcome; do not treat as frozen contract |
| Advanced tier | Use after / instead of basic triage when depth is requested |
| IoPA | Pre-attack indicator - from `/reputation`, not Profile alone |
| IoC | Known compromise feed indicator - from `/reputation` |
| JARM | TLS active fingerprint in `profile.tls.jarm_fingerprint` |
| Redirections | `related_observables` with `related_by: redirects_to` |
| Companion skill | `malanta-api-public` for basic reputation, clusters, batch, history |

---

## 8. Workflows

### Global CHAINED caps

| Resource | Cap |
|----------|-----|
| API calls per Profile chain | 15 |
| Domain/FQDN Profiles | 5 |
| IP Profiles from resolved hosts | 5 (prefer top 3) |
| Auto reputation handoffs | Suggest only unless user opted into classify/deep dive |
| Cache | Reuse Profile per indicator within one investigation |

### WF-P1 Domain capability + attribution snapshot

**Triggers:** profile {domain} / capabilities / services / TLS / web / tags / attribution / advanced enrichment for apex

**ATOMIC:**

```
1. GET /v1/domains/{domain}/profile
2. Present: capabilities (services, tags, web, functions, TLS, redirects) then attribution
3. Briefly note supporting context; suggest /reputation if no prior verdict (do not auto-run)
```

**CHAINED** (user asked profile + classify / deep dive):

```
1. Prefer companion /reputation first if no verdict yet (basic)
2. GET /v1/domains/{domain}/profile   } advanced capabilities + attribution
3. GET /v1/clusters?indicator={domain} } when campaign requested
```

If input is a subdomain, use WF-P2 instead (do not call domains profile).

### WF-P2 FQDN / Host capability snapshot

**Triggers:** profile {fqdn} / host capabilities / subdomain services-web-TLS

```
1. GET /v1/fqdns/{fqdn}/profile
2. Lead host-scoped capabilities + attribution; apex state/registration secondary
3. ATOMIC suggest: apex /reputation if not a shared-platform apex
```

### WF-P3 IP capability snapshot

**Triggers:** profile {ip} / IP services / TLS / tags / attribution on IP

```
1. GET /v1/ips/{ip}/profile
2. Present capabilities (services, tags, TLS, web, redirects) then attribution
3. Suggest IP /reputation (companion) when user wants verdict
```

Do **not** call domain Profile with an IP string.

### WF-P4 Profile → Hosting capability expansion

**Triggers:** profile then expand IPs / compare host capabilities across resolved IPs

```
PRE-CHECK: abort mass expansion on shared-platform / major CDN apex unless user insists on exact hosts
1. GET domain or FQDN /profile (capabilities + attribution first)
2. Extract IPs from profile.network.ip_addresses and related_observables resolves_to
3. GET /v1/ips/{ip}/profile for up to 3 highest-signal IPs (prefer non-CDN when distinguishable)
4. Compare capability surfaces across IPs; suggest reputation/clusters only if classify requested
```

### WF-P5 Capability monitor diff (lightweight)

**Triggers:** re-profile / did capabilities change / new services, JARM, redirects, tags, attribution

```
1. GET matching /profile for the indicator
2. Diff vs prior (if supplied): services ports, tags, JARM/TLS, web/JS capabilities, redirects, intelligence
3. On material capability/attribution change, suggest reputation re-check (companion)
```

Profile has no server-side watchlist API in this skill - client-side diff only. BETA: diff keys may evolve.

### Composition

| From | To |
|------|----|
| WF-P1 with resolved IPs | WF-P4 or WF-P3 |
| WF-P2 apex not shared platform | Suggest companion WF-1 (domain threat assessment) |
| WF-P3 shared_hosting (small set) | Optional FQDN Profiles (cap) |
| Any Profile + "is it malicious?" | Companion reputation ATOMIC |

Max 2 workflow hops in CHAINED mode without user confirmation.

### Chain error handling

| Error | Behavior |
|-------|----------|
| 422 subdomain on `/domains` | Retry `/v1/fqdns/{fqdn}/profile` |
| 400 temporal params | Retry without `history`/`since`/`until` |
| 422 malformed IP | Abort IP step; report invalid input |
| 429 | Pause, backoff, resume |
| 500/504 | Retry once; skip step; continue |
| 401/403 | Abort chain; report auth failure without leaking the key |

---

## 9. Example Scenarios

**A. Advanced capability ask**  
User: "What capabilities does suspicious-login-portal.com expose?"  
→ `GET /v1/domains/suspicious-login-portal.com/profile`  
→ Lead services, tags, web/functions, TLS, redirects, then attribution. Note BETA. Suggest reputation only if no prior verdict.

**B. Subdomain host behavior**  
User: "Profile www.evil-login.com - services, TLS, redirects"  
→ `GET /v1/fqdns/www.evil-login.com/profile` (not `/domains/`).  
→ Host-scoped capabilities + attribution; apex registration secondary.

**C. IP services / TLS**  
User: "What services and TLS fingerprint does 203.0.113.45 show?"  
→ `GET /v1/ips/203.0.113.45/profile` → services + TLS/JARM + tags/attribution.  
→ No `state`/`registration`/`dns` expected.

**D. Classify first, then advanced Profile**  
User: "Is it malicious, then show capabilities and attribution"  
→ Companion `/reputation` first → then Profile for capabilities + attribution.  
→ Keep verdict and capability narratives separate.

**E. Wrong first API**  
User: "Quick check if these 20 domains are IOPA"  
→ Do **not** use Profile. Companion batch `/reputation`. Offer Profile later on HIGH hits for capability depth.

**F. Shared hosting noise**  
Profile returns hundreds of `shared_hosting` on a Cloudflare IP  
→ Report CDN context. Do not treat co-tenants as capability signal. Stay on services/tags/TLS/web/attribution for the queried indicator.

**G. Feedback**  
User hits incomplete web/JS section  
→ State BETA / WIP; share `meta.request_id` with support@malanta.ai; do not invent missing fields.

---

## 10. Integration Checklist

1. Label integrations as **Profile API BETA**; welcome feedback to support@malanta.ai
2. Obtain a Malanta API key; store as a secret
3. Use companion skill for basic triage; call Profile for **advanced** capabilities + attribution
4. Call `https://app.malanta.ai/data/v1/{domains|fqdns|ips}/{x}/profile` with `x-api-key`
5. Present order: services → tags → web/functions → TLS → redirections → attribution → supporting context
6. Default ATOMIC; CHAINED only on explicit deep-dive / expand language
7. Enforce: no temporal params, no batch Profile, eTLD+1 vs FQDN routing
8. Never treat Profile (or JS `dangerous`) as IoPA/IOC verdict
9. Log `meta.request_id` on failures (never log the API key)
10. Use this skill as the agent reference for Malanta Profile API (BETA) work

---

*Malanta Profile API Skill v1.0-public-beta*  
*Structure aligned with Malanta API Skill v4.1-public*  
*BETA - work in progress; feedback welcome: support@malanta.ai*
