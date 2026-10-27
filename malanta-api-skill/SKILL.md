---
name: malanta-api-public
description: >
  Malanta Pre-Attack Prevention via the Malanta API. Use when investigating domains,
  FQDNs, or IP addresses for pre-attack threat intelligence, IoPA/IoC classification,
  attack clusters, campaign mapping, batch triage, watchlist monitoring,
  WHOIS/DNS/certificate enrichment, IP reputation, or ATOMIC/CHAINED investigation
  workflows. Single-file skill for any AI agent or model.
version: 4.1-public
provider: Malanta
category: Pre-Attack Prevention
---

# Malanta API Skill

**Version:** 4.1-public  
**API name:** Malanta API  
**Compatibility:** Any AI agent or model that can call HTTPS APIs and follow structured instructions

Use this skill to query the Malanta API for Pre-Attack Prevention intelligence: classify domains and IP addresses during the adversary setup window, expand attack clusters, and enrich with WHOIS, DNS, and certificates.

---

## 0. Security and Safe Use

### API credentials

- Never print, log, echo, commit, or include API keys in responses or artifacts
- Store the key in the environment or a secrets manager (for example `MALANTA_API_KEY`)
- Auth header: `x-api-key`. Do not put the key in URLs or query strings
- If a key is exposed, rotate it

### Response integrity

- Never fabricate API results, verdicts, scores, APT names, or cluster members
- Empty/`null`/`[]` means no data - say so
- Separate API fields from analytical judgment; use High / Medium / Low confidence on judgments
- `UNKNOWN` means not flagged as attacker infrastructure - not proof the domain or IP is safe

### Handling enrichment data

- Prefer summarizing WHOIS contacts; avoid unnecessary exposure of personal contact details
- Do not pivot on privacy-proxy or redacted WHOIS contacts
- Do not expand shared-platform apexes (see Noise Filtering)

---

## 1. What Malanta Is

Malanta is a Pre-Attack Prevention platform. It identifies and classifies adversary infrastructure during the **setup window** - the gap between when attackers prepare infrastructure and when they strike. Positioning anchors to MITRE ATT&CK **TA0042 (Resource Development)** and **TA0043 (Reconnaissance)**.

| Capability | Meaning |
|-----------|---------|
| **Indicators of Pre-Attack (IoPAs)** | Infrastructure flagged during setup, before attack execution |
| **Attack clusters** | Related domains linked by shared ownership or identity |
| **Setup window detection** | Temporal intelligence between infrastructure creation and attack |
| **Campaign mapping** | Expand from one domain to the operator network via clusters |
| **APT attribution at TA0042** | Threat-actor names from infrastructure identity (`apt_names[]`) |
| **IP reputation** | Classify IPv4 addresses for IoPA/IoC labels, APT, and cluster membership |

**Key terminology:** Adversary Infrastructure Identity, common identity attributes, IoPAs.  
**Category:** Pre-Attack Prevention.

---

## 2. Malanta API Configuration

```
Base URL:  https://app.malanta.ai/data
Version:   /v1
Auth:      x-api-key header (tenant key, typically malanta_ prefix)
Methods:   GET (single) + POST (batch, JSON body)
Response:  { data, query, meta, pagination } envelope
Batch cap: 100 items per POST
```

### Authentication example

```bash
BASE_URL="https://app.malanta.ai/data"
API_KEY="${MALANTA_API_KEY}"

curl -sS "$BASE_URL/v1/domains/example.com/reputation" \
  -H "x-api-key: $API_KEY"

curl -sS "$BASE_URL/v1/ips/203.0.113.45/reputation" \
  -H "x-api-key: $API_KEY"
```

### Response envelope

```json
{
  "data": {},
  "query": {
    "indicator": {
      "type": "domain-name",
      "value": "example.com"
    }
  },
  "meta": {
    "request_id": "uuid",
    "schema_version": "2.0.0",
    "endpoint": "...",
    "elapsed_ms": 142,
    "generated_at": "2026-07-01T12:00:00Z",
    "coverage": null
  },
  "pagination": { "has_more": false, "next_cursor": null }
}
```

- Paginate while `pagination.has_more` is true; pass `pagination.next_cursor` as `?cursor=`
- `meta.request_id` helps Malanta support diagnose failed calls
- `meta.coverage` appears on temporal endpoints (whois, certificates, dns-records)

### Indicator object

| Field | Notes |
|-------|-------|
| `type` | `domain-name`, `fqdn`, `cluster-id`, or `ipv4-addr` |
| `value` | Canonical display value |
| `indicator_id` | Stable UUID when resolvable |
| `ip_addr_hex` | Set for IP indicators; null for domain indicators |

### Input validation (pre-call)

| Check | Rule | On failure |
|-------|------|------------|
| `/v1/domains/` routes | Must be eTLD+1 (registered apex) | Use `/v1/fqdns/` for subdomains |
| `/v1/ips/` reputation | IPv4 address string | Do not pass domains or FQDNs |
| Batch POST | Max 100 items | Chunk requests |
| Cluster `?indicator=` | Domain only | Email/IP return 422 |
| Domain hosting IPs (ASN/geo/ranges) | Use `GET /v1/domains/{domain}/ip-ranges` | Never call `/ipinfo` (path does not exist) |
| Standalone IP reputation | Use `GET /v1/ips/{ip}/reputation` or batch `POST /v1/ips/reputation` | Do not use `/ip-ranges` for this |
| IP geolocation / port scan / CVE by IP | Out of scope for this skill | Use another provider |

**Do not confuse** domain hosting enrichment (`/ip-ranges`, triggered by `available.ipinfo`) with **IP reputation** (`/v1/ips/.../reputation`).

---

## 3. Agent Behavior

### Orchestration modes

| Mode | When | Behavior |
|------|------|----------|
| **ATOMIC** (default) | Normal prompts | One best endpoint. Suggest next steps. Do not auto-chain |
| **CHAINED** | User says "full investigation", "deep dive", "expand", or "map" | Run a workflow sequence. Present progress. Respect caps |

### Core rules

1. Prefer Tier 1 (reputation + clusters) over Tier 2 enrichment unless the user asked for WHOIS/DNS/certs
2. Lead with verdict. Highlight **IOPA** as the pre-attack signal
3. Apply noise filters before expansion
4. Domain routes need eTLD+1; subdomains use `/v1/fqdns/`
5. Raw IPv4 → IP reputation routes. Domain → domain routes. Never invent paths like `/ipinfo`
6. `available.*` on `/info` are boolean flags, not URL paths. Map them to real endpoints (see 5.1)
7. Use the **Malanta API** only as defined here

### Result presentation

**Reputation (domain or IP)**

- Lead with `MALICIOUS` or `UNKNOWN`
- If MALICIOUS: score band (HIGH/MEDIUM/LOW), labels, APT names, cluster count
- **IOPA** = flagged during pre-attack setup window
- **IOC** = confirmed by external threat feeds
- **IOPA + IOC** = highest conviction
- If UNKNOWN: "No attacker-infrastructure link found. This is not evidence of safety."

**Clusters**

- Lead with `cluster_type` (why members are linked) when full cluster detail is fetched
- On IP/domain reputation responses, `clusters[]` may be membership summaries (`cluster_id`, `member_count`, `joined_at`, `reputation`) - suggest `GET /v1/clusters/{cluster_id}` for full detail
- Confidence band, lifecycle, member count, APT if present
- Smaller high-score CONFIRMED clusters are stronger signal than huge low-conviction sets
- Member verdict UNKNOWN inside a MALICIOUS cluster still matters: membership is the signal

### Suggest next steps (ATOMIC only - do not auto-run)

| Current result | Priority | Secondary |
|----------------|----------|-----------|
| MALICIOUS domain | Investigate attack cluster / operator map | WHOIS / DNS |
| MALICIOUS IP | Deep dive top `clusters[]` via cluster-by-id | Batch related IPs if user supplies a list |
| MALICIOUS + clusters | Deep dive cluster / classify members | Enrich seed |
| Batch: many MALICIOUS | Shared campaign? (shared `cluster_id`) | Highest score first |
| IOPA present | Expand related setup infrastructure | Registration timing (domains) |
| UNKNOWN | Re-check later / other providers | - |

---

## 4. Prompt-to-Endpoint Map (ATOMIC)

### Tier 1 - Reputation and clusters

| User intent | Endpoint |
|-------------|----------|
| What do we know / quick check (domain) | `GET /v1/domains/{domain}/info` |
| Full domain reputation / APT / pre-attack | `GET /v1/domains/{domain}/reputation` |
| Full IP reputation / APT / pre-attack | `GET /v1/ips/{ip}/reputation` |
| Batch classify domains / which are IOPA | `POST /v1/domains/reputation` body `{"domains":[...]}` |
| Batch classify IPs / which are IOPA | `POST /v1/ips/reputation` body `{"ips":[...]}` |
| Campaign / related attacker infra (domain seed) | `GET /v1/clusters?indicator={domain}` |
| Cluster by id | `GET /v1/clusters/{cluster_id}` |
| Sender domains from email / DNS log destinations | Domain reputation GET or POST after extraction |
| Firewall / proxy / SIEM IPs | IP reputation GET or POST |

### Tier 2 - Enrichment

| User intent | Endpoint |
|-------------|----------|
| WHOIS / history | `GET /v1/domains/{domain}/whois` (`?history=true`, optional `?since=` / `?until=`) |
| DNS | `GET /v1/domains/{domain}/dns-records` (`?record_type=`, `?since=` / `?until=`) |
| Certificates | `GET /v1/domains/{domain}/certificates` (`?history=true`, `?limit=`, temporal) |
| Hosting IPs / IP ranges for a domain | `GET /v1/domains/{domain}/ip-ranges` (batch: `POST /v1/domains/ip-ranges`) |
| Code repos | `GET /v1/domains/{domain}/code-repos` |
| Subdomain DNS/certs/repos | `GET /v1/fqdns/{fqdn}/dns-records` or `/certificates` or `/code-repos` |

**Do not call** `/v1/domains/{domain}/ipinfo` or any `/ipinfo` path. That endpoint does not exist. When `/info` returns `available.ipinfo: true`, call **`/ip-ranges`** (hosting enrichment), not IP reputation.

### Out of scope for this skill

Malware sandboxing, file hashes, CVE lookup, email header parsing, PCAP decode, URL screenshots, dark web monitoring, IP geolocation-only lookups, port scans. Parse elsewhere, then enrich extracted domains or IPs with Malanta reputation where applicable.

---

## 5. Execution Reference

### 5.1 GET /v1/domains/{domain}/info

Triage: verdict + `available.*` existence flags + observation timestamps.  
Use `/reputation` when you need labels, APT, IoC sources, cluster membership detail.

**`available` flags are not endpoints.** They only mean "this related route has data." Map flags to calls:

| `available` flag | Call this endpoint when true | Do not call |
|------------------|------------------------------|-------------|
| `whois` | `GET /v1/domains/{domain}/whois` | - |
| `certificates` | `GET /v1/domains/{domain}/certificates` | - |
| `clusters` | `GET /v1/clusters?indicator={domain}` | - |
| `dns_records` | `GET /v1/domains/{domain}/dns-records` | - |
| `co_hosted_domains` | `GET /v1/domains/{domain}/co-hosted-domains` | - |
| `ipinfo` | `GET /v1/domains/{domain}/ip-ranges` | **`/ipinfo` (does not exist)** |
| `code_repos` | `GET /v1/domains/{domain}/code-repos` | - |

### 5.2 GET /v1/domains/{domain}/reputation

Full domain classification.

| Field | Description |
|-------|-------------|
| `reputation.verdict` | `MALICIOUS` or `UNKNOWN` |
| `reputation.labels` | `IOPA`, `IOC`, or both |
| `reputation.malicious_score` | 0.0-1.0 or null |
| `reputation.malicious_score_band` | `HIGH`, `MEDIUM`, `LOW`, or null |
| `apt_names` | Threat-actor attribution (no separate APT endpoint) |
| `ioc_sources` | Feed / source names |
| `clusters[]` | `{ cluster_id, member_count, joined_at, reputation }` |
| `timestamps` | `first_observed_at`, `last_classified_at`, `last_observed_at` |

**Batch:** `POST /v1/domains/reputation` with `{"domains":["a.com","b.com"]}` (max 100). `data` is an array in input order. Batch is current-state only (no temporal params).

### 5.3 GET /v1/ips/{ip}/reputation

Full **IP reputation** classification for a single IPv4 address.

```bash
curl -sS "$BASE_URL/v1/ips/203.0.113.45/reputation" \
  -H "x-api-key: $API_KEY"
```

| Field | Description |
|-------|-------------|
| `indicator.type` | `ipv4-addr` |
| `indicator.value` | Queried IPv4 string |
| `indicator.ip_addr_hex` | Hex encoding of the address (when present) |
| `reputation.verdict` | `MALICIOUS` or `UNKNOWN` |
| `reputation.labels` | `IOPA`, `IOC`, or both |
| `reputation.malicious_score` | 0.0-1.0 or null |
| `reputation.malicious_score_band` | `HIGH`, `MEDIUM`, `LOW`, or null |
| `apt_names` | Threat-actor attribution |
| `ioc_sources` | Feed / source names |
| `context` | Additional context array (may be empty) |
| `domain_status` | Often null on IP responses |
| `clusters[]` | Membership summaries: `{ cluster_id, member_count, joined_at, reputation }` |
| `timestamps` | `first_observed_at`, `last_classified_at`, `last_observed_at` |

**Batch:** `POST /v1/ips/reputation` with `{"ips":["1.2.3.4","5.6.7.8"]}` (max 100).

```bash
curl -sS -X POST "$BASE_URL/v1/ips/reputation" \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"ips":["203.0.113.45","198.51.100.10"]}'
```

- `data` is an array of per-IP objects (same fields as single GET `data`, plus embedded `indicator`)
- Batch is current-state only (no temporal params)
- `query.indicator` may be null on batch responses
- Prefer batch when the user supplies multiple IPs

**Criteria for choosing IP reputation**

| Input / prompt | Call |
|----------------|------|
| Single IPv4 ("is this IP malicious?") | `GET /v1/ips/{ip}/reputation` |
| List of IPv4s / batch triage / watchlist of IPs | `POST /v1/ips/reputation` |
| Domain hosting / ASN / geo for a domain | `GET /v1/domains/{domain}/ip-ranges` (not IP reputation) |
| Domain classification | Domain `/info` or `/reputation` |

### 5.4 GET /v1/clusters?indicator={domain}

Returns attack clusters for a **domain**. Optional `?malicious=true`, `?limit=`, `?cursor=`.  
Cluster lookup by IP indicator is not supported on this route (email/IP return 422). For IPs, use `clusters[]` from IP reputation, then `GET /v1/clusters/{cluster_id}`.

**Cluster object (high level):** `cluster_id`, `cluster_class`, `cluster_type` (`SHARED_REGISTRANT_CLUSTER` | `OWNERSHIP_TREE_CLUSTER` | `SHARED_CERTIFICATE_CLUSTER`), `confidence` / `confidence_band`, `lifecycle_status`, `member_count`, `members_truncated`, timestamps, `reputation`, `apt_names`, `ioc_sources`, `members[]`.

**Member object:** `indicator`, `joined_at`, observation times, per-member `reputation`, `apt_names`, `ioc_sources`.

### 5.5 GET /v1/clusters/{cluster_id}

Full cluster by id. **404** if not found (treat as missing; do not invent members).  
`cluster_id` is a stable join key across investigations (STIX: `intrusion-set--<cluster_id>`).  
Works for cluster IDs discovered from domain or IP reputation responses.

### 5.6 Enrichment

**WHOIS** `GET /v1/domains/{domain}/whois`  
Params: `?history=true`, `?since=`, `?until=`.  
Batch POST current-only.  
Fields: registrar, created/updated/expires, statuses, name_servers, contacts (REGISTRANT/ADMINISTRATIVE/TECHNICAL/BILLING), `history[]` with periods.

**Certificates** `GET /v1/domains/{domain}/certificates`  
Params: `?history=true`, `?limit=` (default 100), temporal. Paginate if `has_more`.  
Fields: serials, validity, subject (CN, org, SANs), issuer, self-signed flag, fingerprints, observation period.

**DNS** `GET /v1/domains/{domain}/dns-records`  
Params: `?record_type=`, `?since=`, `?until=`, cursor pagination.  
Fields: `record_type`, `value`, `target`, `ttl`, `period`.

**IP ranges (domain hosting)** `GET /v1/domains/{domain}/ip-ranges`  
Current-only. Cursor-paged. Returns IPs the domain is hosted on, each with observed window plus network block (start/end, CIDR), company, ASN, geo, `ip_type`.  
**Batch:** `POST /v1/domains/ip-ranges` with `{"domains":[...]}` (max 100), current-only.  
Triggered by `available.ipinfo: true` on `/info`. **Never** translate that flag into `/ipinfo`.  
Optional follow-up: for each returned hosting IP the user wants classified, call IP reputation (`/v1/ips/...`).

**Code repos** `GET /v1/domains/{domain}/code-repos`  
Repo is the flagged side. Benign domains appear in security tooling lists. Check `occurrence.validation_status` and evidence context.

**FQDN routes** (no eTLD+1 restriction):  
`/v1/fqdns/{fqdn}/certificates`, `/dns-records`, `/code-repos` (+ batch POST variants with `{"fqdns":[...]}`).

### 5.7 Errors

| Status | Action |
|--------|--------|
| 200 | Success; empty data is not an error |
| 400 | Fix params / batch size / temporal usage |
| 401/403 | Auth failure; verify key; include `request_id` for support |
| 404 | Not found (by-id clusters) |
| 422 | Bad input (not eTLD+1, wrong indicator type) |
| 429 | Back off exponentially |
| 500 / 504 | Retry briefly; then skip step in a chain and note gap |

Batch POST rejects temporal params (`history`, `since`, `until`) with 400.

---

## 6. Noise Filtering

- Skip major provider apexes for expansion (google.com, cloudflare.com, amazonaws.com) unless specifically targeted
- Shared-platform apexes (vercel.app, pages.dev, github.io, workers.dev, netlify.app, icp0.io, URL shorteners): do **not** cluster-expand the platform apex
- Very large clusters (10,000+ members): weak evidence for any single member; prefer small high-score CONFIRMED/HIGH clusters
- Never pivot on privacy-proxy WHOIS (WhoisGuard, Withheld for Privacy, Domains By Proxy, and similar)
- Prefer server-side `SHARED_REGISTRANT_CLUSTER` / ownership clusters over manual email matching
- Skip freemail apex pivots (gmail.com, outlook.com, yahoo.com, and similar)
- Code-repos: keep shallow pivots; repo is flagged, not necessarily the domain
- Dedupe and normalize inputs before batching
- IP reputation: still report the API verdict for any IPv4 the user asked about. When expanding from IP `clusters[]`, prefer small high-score clusters. Do not invent IP geolocation from reputation alone.

---

## 7. Concepts Quick Reference

| Term | Meaning |
|------|---------|
| IoPA | Pre-attack indicator (setup window) |
| IoC | Known compromise feed indicator |
| Setup window | Time between infra acquisition and attack |
| Attack cluster | Related domains by ownership / registrant / certificate identity |
| Cluster risk | (MALICIOUS members) / total members |
| Score bands | HIGH / MEDIUM / LOW (and CONFIRMED at cluster/high-conviction levels) |
| IP reputation | Malanta classification of an IPv4 via `/v1/ips/.../reputation` |
| IP ranges | Domain hosting / ASN / geo enrichment via `/ip-ranges` (not reputation) |

---

## 8. Workflows

### Global CHAINED caps

| Resource | Cap |
|----------|-----|
| API calls per chain | 20 |
| Unique domains enriched | 200 |
| Unique IPs enriched | 200 |
| Cluster deep dives | 5 (prefer top 3 by score) |
| WHOIS on members | 5 highest-score |
| Cache | Reuse reputation per indicator within one investigation |

### WF-1 Domain Threat Assessment

**Triggers:** investigate / threat level / full assessment / deep dive {domain}

**ATOMIC:** info → reputation → clusters → (optional) whois history → dns  

**CHAINED:**

```
1. GET /v1/domains/{domain}/info
2. GET /v1/domains/{domain}/reputation
3. GET /v1/clusters?indicator={domain}
4. GET /v1/domains/{domain}/whois?history=true   } parallel
5. GET /v1/domains/{domain}/dns-records         }
Optional: certificates?history=true when available.certificates is true
Optional: ip-ranges when available.ipinfo is true; then IP reputation on user-selected hosting IPs
```

Highlight IOPA. If user only asked "is it malicious?" and verdict is UNKNOWN, stop after info/reputation.

### WF-2 Infrastructure Expansion

**Triggers:** expand / map campaign / what else is connected

```
PRE-CHECK: abort if shared-platform apex
1. GET domain reputation
2. GET clusters?indicator=
3. POST /v1/domains/reputation for members (100/batch, respect 200 domain cap)
4. GET /v1/clusters/{id} for top clusters (cap 3)
```

### WF-3 Cluster Deep Dive

**Triggers:** deep dive cluster {id} / entire campaign

```
1. GET /v1/clusters/{cluster_id}
2. POST batch reputation on domain members
3. Risk = MALICIOUS / total; WHOIS on up to 5 HIGH members
4. Correlate apt_names from cluster + members
```

If cluster returns 404: report unresolved id; do not invent membership.

### WF-4 Batch Triage

```
Domains:
1. POST /v1/domains/reputation (chunk 100)
2. Group: IOPA-only, IOC-only, IOPA+IOC, UNKNOWN; sort MALICIOUS by band
3. Optional: cluster lookup for up to 5 HIGH-band domain hits; detect shared cluster_id

IPs:
1. POST /v1/ips/reputation (chunk 100)
2. Group the same way
3. Optional: GET /v1/clusters/{id} for up to 5 HIGH-band hits' top cluster_ids
```

Do not mix domains and IPs in one POST body. Split mixed lists into domain batch + IP batch.

### WF-5 Watchlist Monitoring

```
Domain watchlist: POST /v1/domains/reputation
IP watchlist: POST /v1/ips/reputation
Diff vs prior: UNKNOWN→MALICIOUS, new labels, new apt_names, new clusters
Use timestamps.last_classified_at for change detection
On new MALICIOUS domain: run WF-1 / WF-2 with user confirmation
On new MALICIOUS IP: run WF-10 with user confirmation
```

### WF-6 Cross-Investigation Correlation

Match `cluster_id` across cases (including IDs from IP reputation). Same id = same operator infrastructure. Re-fetch `GET /v1/clusters/{id}` for current membership.

### WF-7 Email-Extracted Domain Enrichment

Prerequisite: sender domains already extracted (Malanta does not parse email).  
Then WF-4 / reputation + optional clusters. Emphasize IOPA = phishing infra flagged before send.

### WF-8 Registrant Pivot

Prefer cluster `SHARED_REGISTRANT_CLUSTER` over manual WHOIS. Skip privacy and freemail pivots.

### WF-9 FQDN / Subdomain

```
1. GET /v1/fqdns/{fqdn}/dns-records (and/or certificates)
2. If apex is not a shared platform: run WF-1 on eTLD+1
```

### WF-10 IP Threat Assessment

**Triggers:** investigate IP / is this IP malicious / IP reputation / deep dive {ip} / check these IPs

**ATOMIC (single IP):**

```
1. GET /v1/ips/{ip}/reputation
2. Suggest: GET /v1/clusters/{cluster_id} for top memberships (do not auto-run unless CHAINED)
```

**ATOMIC (list of IPs):**

```
1. POST /v1/ips/reputation {"ips":[...]} (chunk 100)
2. Group IOPA / IOC / both / UNKNOWN; lead with MALICIOUS HIGH
```

**CHAINED (single IP deep dive):**

```
1. GET /v1/ips/{ip}/reputation
2. GET /v1/clusters/{id} for up to 3 highest-score clusters from clusters[]
3. Optional: POST /v1/domains/reputation on domain members from those clusters (respect domain cap)
```

Highlight IOPA vs IOC. If user only asked "is it malicious?" and verdict is UNKNOWN, stop after step 1.  
Do **not** call `/ip-ranges` for a bare IP. Do **not** call domain routes with an IP string.

### Composition

| From | To |
|------|----|
| WF-1 MALICIOUS + clusters | WF-2 or WF-3 |
| WF-2 cluster found | WF-3 |
| WF-4 HIGH domain hits | WF-1 |
| WF-4 HIGH IP hits | WF-10 |
| WF-5 new MALICIOUS domain | WF-1 + WF-2 |
| WF-5 new MALICIOUS IP | WF-10 |
| WF-7 MALICIOUS senders | WF-2 |
| WF-9 apex ok | WF-1 |
| WF-10 MALICIOUS + clusters | WF-3 |

Max 2 workflow hops in CHAINED mode without user confirmation.

### Chain error handling

| Error | Behavior |
|-------|----------|
| 422 not eTLD+1 | Retry FQDN route if applicable; else skip |
| 422 cluster indicator is IP | Use IP reputation `clusters[]` + cluster-by-id instead |
| 429 | Pause, backoff, resume |
| 404 cluster | Skip; note gap |
| 500/504 | Retry once; skip step; continue |
| 401/403 | Abort chain; report auth failure without leaking the key |

---

## 9. Example Scenarios

**A. Pre-attack hit**  
User: "Is suspicious-login-portal.com a threat?"  
→ info/reputation → MALICIOUS HIGH labels `[IOPA]` → explain setup-window detection → suggest clusters.

**B. Campaign map**  
User: "Map infrastructure from evil-login.com"  
→ WF-2 CHAINED → reputation → clusters → batch members → top cluster detail.

**C. Batch triage**  
User: "Check these 20 brand alerts"  
→ POST domain reputation → group IOPA / IOC / both → suggest shared-cluster check.

**D. IP reputation**  
User: "Is 203.0.113.45 malicious?"  
→ `GET /v1/ips/203.0.113.45/reputation` → lead with verdict / labels / APT / clusters.  
User: "Check these firewall IPs"  
→ `POST /v1/ips/reputation` with `{"ips":[...]}` → WF-4 IP path.  
User: `/info` shows `available.ipinfo: true`  
→ Call `GET /v1/domains/{domain}/ip-ranges`. Do **not** call `/ipinfo`. Optionally classify returned hosting IPs with IP reputation.

**E. Shared platform**  
User: "Expand test-page.vercel.app"  
→ FQDN enrichment only. Do not expand vercel.app apex.

**F. Out of scope**  
User: "What CVEs hit this host?" or "Geo-locate this IP only"  
→ State out of scope for CVE / geo-only. Offer IP reputation if classification is useful.

---

## 10. Integration Checklist

1. Obtain a Malanta API key; store as a secret
2. Call `https://app.malanta.ai/data/v1/...` with `x-api-key`
3. Default ATOMIC; enable CHAINED only on explicit request
4. Enforce batch size 100 and chain caps (domains and IPs)
5. Route IPv4 to `/v1/ips/.../reputation`; route domains to `/v1/domains/...`
6. Keep `/ip-ranges` for domain hosting only; never invent `/ipinfo`
7. Log `meta.request_id` on failures (never log the API key)
8. Use this skill as the agent reference for Malanta API work

---

*Malanta API Skill v4.1-public*
