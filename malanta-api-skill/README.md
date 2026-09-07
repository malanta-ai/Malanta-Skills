# Malanta API Skill

**Skill name:** `malanta-api-public`  
**Version:** 4.0-public  
**API:** Malanta API (`https://app.malanta.ai/data/v1`)

Single-file agent skill for Pre-Attack Prevention investigations: domain reputation, IoPA/IoC labels, attack clusters, campaign expansion, WHOIS/DNS/certificate enrichment, and ATOMIC / CHAINED workflows.

## Files in this folder

| File | Use |
|------|-----|
| [`SKILL.md`](./SKILL.md) | **Canonical** skill file. Point agents at this. |
| [`malanta-api-public-v40-SKILL.md`](./malanta-api-public-v40-SKILL.md) | Same content with an explicit versioned filename (handy for packaging or pinning). |

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
# Use the env var name your Malanta integration documents
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
2. Tell the agent: “Follow the Malanta API Skill for all Malanta investigations.”
3. Give the agent a tool or shell that can run authenticated HTTP requests.
4. Inject `MALANTA_API_KEY` (or equivalent) into the tool environment only.

Works with: custom RAG agents, LangGraph/Crew-style runners, OpenAI Assistants system prompt, Anthropic system prompt, Gemini system instruction, etc.

### Pattern B - Cursor Agent Skills

1. Copy this folder into your project:

```text
.cursor/skills/malanta-api-public/SKILL.md
```

2. Or copy into your user skills:

```text
~/.cursor/skills/malanta-api-public/SKILL.md
```

3. Ensure the YAML frontmatter `name` / `description` remain intact (Cursor uses them for discovery).
4. In chat, invoke by name or describe a Malanta investigation (domain check, deep dive, cluster map).
5. Put the API key in the project `.env` (gitignored) or OS env; do not commit it.

### Pattern C - Claude Projects / Custom GPTs / similar

1. Upload `SKILL.md` as project knowledge / instructions file **or** paste it into custom instructions.
2. Enable web/API or code-execution tools if the product supports outbound HTTPS.
3. Add a short instruction: “When investigating domains or campaigns with Malanta, follow SKILL.md exactly. Use ATOMIC by default; CHAINED only on deep dive / expand / map.”

### Pattern D - CI / automation harness

1. Check out this repo (or vendor the `malanta-api-skill/` folder).
2. At job start, load `SKILL.md` into the agent prompt builder.
3. Mount secrets from your vault as `MALANTA_API_KEY`.
4. Cap concurrency and respect skill chain limits (batch ≤ 100, chain caps in the skill).

### Pattern E - MCP or tool-router setups

1. Keep `SKILL.md` as the **policy / routing** layer for the LLM.
2. Implement thin tools that mirror Malanta routes (`domains/reputation`, `clusters`, etc.) if you prefer structured tool calling over raw `curl`.
3. The skill still owns: when to call what, ATOMIC vs CHAINED, noise filters, and no invented paths (for example never `/ipinfo` - use `/ip-ranges` when `available.ipinfo` is true).

---

## Minimal verification

After wiring the skill, run a single ATOMIC check:

```bash
curl -sS "https://app.malanta.ai/data/v1/domains/example.com/info" \
  -H "x-api-key: $MALANTA_API_KEY"
```

Expect a JSON envelope `{ data, query, meta, pagination }`. Then ask the agent: “Using the Malanta API Skill, check example.com” and confirm it calls `/info` or `/reputation` (not invented paths).

---

## Behavior summary (see SKILL.md for full detail)

- **ATOMIC** (default): one best endpoint; suggest next steps
- **CHAINED**: only when the user asks for full investigation / deep dive / expand / map
- Tier 1 first: reputation + clusters; Tier 2: WHOIS / DNS / certificates / ip-ranges / code-repos
- Domain routes need eTLD+1; subdomains use `/v1/fqdns/`
- `available.*` on `/info` are flags, not URL segments (`ipinfo` flag → `/ip-ranges`)

---

## Docs

- Malanta API help: https://help.malanta.ai/en/article/apis-1mtukvh
- Repo overview: [../README.md](../README.md)
