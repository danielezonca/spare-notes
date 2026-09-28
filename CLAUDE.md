# CLAUDE.md

## About this repo

Design and analysis repo — no code. Architecture decisions, migration
plans, gap assessments, and research for the AI Gateway platform
(Grid, MaaS, Pricetag).

Folders:
- `grid-mvp/` — multi-cluster Grid architecture
- `tokenomics/` — token rate limiting, metering, billing
- `pricetag/` — Pricetag dogfood migration and product alignment

## Rules

### No customer or internal data

**NEVER include any of the following in committed files:**
- Customer names, company names, or person names
- Internal Jira keys (RHAIRFE-*, RHAISTRAT-*, etc.)
- Internal Jira/Confluence/Atlassian URLs
- Internal system names that identify specific customers
- Meeting attendees or participant lists
- Customer-specific project IDs or account identifiers

If customer-sourced material is needed for analysis, place it in an
`other_sources/` subfolder — these are gitignored repo-wide.

Use generic terms: "customer", "Engineering Lead", "the onboarding
service", "internal tracking".

### Source attribution

Every analysis document MUST include links to sources:
- GitHub repo/issue/PR URLs
- File paths within analyzed repos
- Document names when repos aren't linkable

A claim without a source is an opinion, not analysis.

### Conventions

- Markdown for decisions, comparisons, gaps
- HTML/SVG for deployment diagrams (dark theme)
- Each top-level folder has a `README.md` index
- `other_sources/` subfolders are gitignored
- Cross-reference between files instead of duplicating

## Agent guidelines

See [AGENTS.md](AGENTS.md) for agent-specific rules on source
attribution, data safety, output format, and repo context.
