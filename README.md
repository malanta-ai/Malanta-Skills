# Malanta Skills

Public and shareable **agent skills** for Malanta Pre-Attack Prevention.

This repository packages skills that any AI agent, IDE harness, or automation runner can load to work with Malanta products and APIs correctly: routing, endpoint contracts, investigation workflows, and safe handling of credentials and enrichment data.

## What’s included

| Skill folder | Purpose | Version |
|--------------|---------|---------|
| [`malanta-api-skill/`](./malanta-api-skill/) | Malanta API investigation skill (domains, clusters, IoPA/IoC, enrichment) | 4.0-public |

More Malanta skills may be added here over time (one folder per skill).

## Who this is for

- Customers and prospects integrating Malanta into agent workflows
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

1. Clone this repository (private / internal access required).
2. Open the skill folder you need (start with [`malanta-api-skill/`](./malanta-api-skill/)).
3. Follow that folder’s README to attach `SKILL.md` to your agent or harness.
4. Supply a Malanta API key via environment / secrets manager. Never commit keys.

## Security

- Do not commit API keys, `.env` files, or customer investigation dumps
- Skills instruct agents never to print or log credentials
- Treat investigation outputs according to your organization’s data-handling policy

## Support

- Malanta API docs: [help.malanta.ai](https://help.malanta.ai)
- Product site: [malanta.ai](https://malanta.ai)

## License / distribution

Internal Malanta distribution for customers, prospects, and partners. Redistribution outside authorized channels requires Malanta approval.
