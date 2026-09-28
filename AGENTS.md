# AGENTS.md

## Agent guidelines for this repo

### Purpose

This repo is a design and analysis workspace. Agents are used to:
- Analyze codebases and extract architecture details
- Cross-reference requirements across multiple sources
- Review documents for consistency, leaks, and gaps
- Generate SVG diagrams and comparison tables

### Source requirements

Every analysis produced by an agent MUST:

1. **Link to sources** — include GitHub repo URLs, issue links, or
   file paths for every factual claim. Use tables like:
   ```
   | Source | Link |
   |--------|------|
   | Praxis AI repo | https://github.com/praxis-proxy/ai |
   | Epic #656 | https://github.com/praxis-proxy/ai/issues/656 |
   ```

2. **Name the documents analyzed** — if reading local files, state
   which files were read (e.g., "from `models-as-a-service/maas-api/`")

3. **Distinguish fact from inference** — if a conclusion is drawn from
   code analysis vs explicitly documented, say so

### Data safety

Agents MUST NOT include in their output:
- Customer names, company names, or person names
- Internal Jira keys or URLs (RHAIRFE-*, RHAISTRAT-*, redhat.atlassian.net)
- Internal system names that identify specific customers
- Meeting attendee lists

If an agent reads customer-sourced material (from `other_sources/`),
it must anonymize before producing output:
- Person names → role-based identifiers ("Engineering Lead")
- Company names → "customer"
- Jira keys → generic identifiers ("RFE-1", "Internal")

### Output format

- Structured markdown with tables for comparisons
- One flat list per extraction task (not nested prose)
- Include an ID column for cross-referencing between analyses
- Use categories consistently across documents:
  `rate-limiting`, `metering`, `auth`, `routing`, `observability`,
  `quota-management`, `billing`, `key-management`, `model-listing`,
  `credential-injection`, `infra`, `migration`, `ui`

### Repo context

Key upstream repos that agents may analyze:
- `praxis-proxy/ai` — AI gateway (Rust, filters, Praxis AI binary)
- `praxis-proxy/grid` — Grid control plane (Rust, CRDs, SWIM, overlay)
- `praxis-proxy/praxis` — Core proxy framework (Rust, Pingora-based)
- `praxis-proxy/experimental` — Demos and prototypes
- `models-as-a-service` (opendatahub-io) — MaaS platform (Go, CRDs, maas-api)
- `ai-gateway-metering-service` — Pricetag metering (Go, PostgreSQL)
