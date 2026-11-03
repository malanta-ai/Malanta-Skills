# Malanta Profile API Skill (BETA)

**Skill name:** `malanta-profile-api-public`  
**Version:** 1.0-public-beta  
**Status:** BETA - work in progress. Feedback welcome: support@malanta.ai  
**API:** Malanta Profile API / Gold Query (`https://app.malanta.ai/data/v1/.../profile`)  
**Tier:** Advanced (capabilities + attribution)  
**Structure:** Aligned with Malanta API Skill v4.1-public section layout

Single-file agent skill for the **advanced** Observable Profile surface. Unique value vs other Malanta APIs:

- **Capabilities:** services, tags, web, JS functions/capabilities, TLS/JARM, redirections
- **Attribution:** `intelligence` APT and categorized attributes

Not for first-pass "is it malicious?" triage. Use the companion skill for reputation, IoPA/IoC, clusters, batch, and history - then escalate to Profile for capability depth.

**Companion:** [`malanta-api-skill/`](../malanta-api-skill/) (`malanta-api-public`).

## Files in this folder

| File | Use |
|------|-----|
| [`SKILL.md`](./SKILL.md) | **Canonical** skill file. Point agents at this. |
| [`malanta-profile-api-public-v10-SKILL.md`](./malanta-profile-api-public-v10-SKILL.md) | Same content with an explicit versioned filename (handy for packaging or pinning). |

Load **one** of them (prefer `SKILL.md`). Do not load both into the same agent context.

---

## Prerequisites

1. A Malanta API tenant key (typically `malanta_` prefix).
2. Network access to `https://app.malanta.ai/data`.
3. An agent runtime that can:
   - Read a Markdown skill / system prompt
   - Call HTTPS APIs (`curl`, SDK, or tool-calling)

Store the key as a secret, for example:

```bash
export MALANTA_API_KEY="malanta_xxxxxxxx"
```

Never put the key in the skill file, prompts committed to git, or chat logs.

Auth header on every call:

```http
x-api-key: <your-key>
```

---

## How to add this skill to any agent or harness

### Pattern A - System / developer prompt (any model)

1. Copy the full contents of `SKILL.md` into the agent’s **system** or **developer** instructions.
2. Tell the agent: “Follow the Malanta Profile API Skill (BETA) for advanced capability and attribution Profile calls. Use malanta-api-public for basic triage.”
3. Give the agent a tool or shell that can run authenticated HTTP requests.
4. Inject `MALANTA_API_KEY` into the tool environment only.

### Pattern B - Cursor Agent Skills

1. Copy this folder into your project:

```text
.cursor/skills/malanta-profile-api-public/SKILL.md
```

2. Or copy into your user skills:

```text
~/.cursor/skills/malanta-profile-api-public/SKILL.md
```

3. Ensure the YAML frontmatter `name` / `description` remain intact.
4. In chat, invoke for capabilities, services, TLS/JARM, web/JS, redirects, tags, or attribution - not for first-pass IoPA checks.
5. Put the API key in the project `.env` (gitignored) or OS env; do not commit it.

### Pattern C - Claude Projects / Custom GPTs / similar

1. Upload `SKILL.md` as project knowledge / instructions **or** paste into custom instructions.
2. Enable outbound HTTPS tools if the product supports them.
3. Add: “Profile is BETA and advanced. Lead with capabilities and attribution. Use the companion Malanta API skill for reputation and clusters.”

### Pattern D - CI / automation harness

1. Check out this repo (or vendor the `malanta-profile-api-skill/` folder).
2. At job start, load `SKILL.md` into the agent prompt builder.
3. Mount secrets from your vault as `MALANTA_API_KEY`.
4. Cap concurrency; respect skill chain limits (no batch Profile; chain caps in the skill).
5. Mark pipelines that depend on Profile as BETA / subject to change.

### Pattern E - MCP or tool-router setups

1. Keep `SKILL.md` as the **policy / routing** layer for advanced Profile.
2. Implement thin tools for `domains/profile`, `fqdns/profile`, `ips/profile` if you prefer structured tool calling over raw `curl`.
3. The skill still owns: advanced-vs-basic routing, capability-first presentation, no temporal params, noise filters, and handoffs to reputation/clusters.

---

## Minimal verification

```bash
curl -sS "https://app.malanta.ai/data/v1/domains/example.com/profile" \
  -H "x-api-key: $MALANTA_API_KEY"

curl -sS "https://app.malanta.ai/data/v1/fqdns/www.example.com/profile" \
  -H "x-api-key: $MALANTA_API_KEY"

curl -sS "https://app.malanta.ai/data/v1/ips/203.0.113.45/profile" \
  -H "x-api-key: $MALANTA_API_KEY"
```

Expect JSON envelope `{ data, query, meta }` with **no** `pagination`. Ask the agent: “Using the Malanta Profile API Skill (BETA), what capabilities and attribution does example.com show?” Confirm it calls `/profile` and leads with services/tags/web/TLS/redirects/intelligence - not `/reputation` as the Profile answer.

---

## Behavior summary (see SKILL.md for full detail)

- **BETA** - WIP; feedback welcome at support@malanta.ai
- **Advanced tier** - after basic triage, or when user asks for capabilities / attribution
- **Lead:** services, tags, web/functions, TLS, redirections, then attribution
- **ATOMIC** default; **CHAINED** only on explicit deep dive / expand
- Domain eTLD+1 → `/v1/domains/.../profile`; subdomain → `/v1/fqdns/.../profile`; IP → `/v1/ips/.../profile`
- No `history` / `since` / `until` (400); no batch POST
- Does **not** replace `/reputation` or `/clusters`

---

## Docs

- Profile API integration guide: https://help.malanta.ai/en/article/profile-api-integration-guide-1akmgbq/
- Malanta API help: https://help.malanta.ai/en/article/apis-1mtukvh
- Companion skill: [../malanta-api-skill/](../malanta-api-skill/)
- Repo overview: [../README.md](../README.md)
- Feedback: support@malanta.ai
