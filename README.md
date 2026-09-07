# Malanta Skills

Public **agent skills** for Malanta Pre-Attack Prevention.

This repository packages skills that any AI agent, IDE harness, or automation runner can load to work with Malanta products and APIs correctly: routing, endpoint contracts, investigation workflows, and safe handling of credentials and enrichment data.

## What’s included

| Skill folder | Purpose | Version |
|--------------|---------|---------|
| [`malanta-api-skill/`](./malanta-api-skill/) | Malanta API investigation skill (domains, clusters, IoPA/IoC, enrichment) | 4.0-public |

More Malanta skills may be added here over time (one folder per skill).

## Who this is for

- Customers and partners integrating Malanta into agent workflows
- Security engineers wiring Malanta into Cursor, Claude, Copilot-style agents, or custom harnesses
- Partners packaging Pre-Attack Prevention into their own tooling

## Repository layout

```
Malanta Skills/
├── README.md                 # This file
└── malanta-api-skill/
    ├── README.md             # How to install and use this skill
    ├── SKILL.md              # Canonical skill definition (load this)
    └── malanta-api-public-v40-SKILL.md  # Same content, versioned filename
```

## Quick start

1. Clone or download this repository (public, read-only).
2. Open the skill folder you need (start with [`malanta-api-skill/`](./malanta-api-skill/)).
3. Follow that folder’s README to attach `SKILL.md` to your agent or harness.
4. Supply a Malanta API key via environment / secrets manager. Never commit keys.

```bash
git clone https://github.com/malanta-ai/Malanta-Skills.git
```

## Distribution policy (read-only)

This repository is a **distribution source only** for prospects, customers, and partners.

- Clone and download are welcome
- External contributions are **not** accepted (no issues, discussions, forks, or pull requests from outside Malanta)
- Only Malanta org members can change the repository
- Treat investigation outputs according to your organization’s data-handling policy

## Security

- Do not commit API keys, `.env` files, or customer investigation dumps
- Skills instruct agents never to print or log credentials
- Report suspected key exposure to Malanta support; rotate the key

## Support

- Malanta API docs: [help.malanta.ai](https://help.malanta.ai)
- Product site: [malanta.ai](https://malanta.ai)

## License / distribution

Published by Malanta Ltd. for customer, prospect, and partner use with the Malanta API. Skills document the public Malanta API surface only. Redistribution of modified copies as official Malanta materials requires Malanta approval.
