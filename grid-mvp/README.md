# AI Grid MVP

Architecture design for multi-cluster AI Grid deployment — routing
inference across clusters with MaaS (Models-as-a-Service) integration.

## Documents

| Document | Description |
|----------|------------|
| [Design Notes](design-notes.md) | Architecture decisions, migration strategy, component overview |
| [Gaps](gaps.md) | Changes required to existing components (maas-api, Praxis AI, Grid Operator, infra) |
| [Auth & Rate Limiting](auth-and-ratelimit.md) | Detailed auth and rate limiting flows across Grid and MaaS layers |
| [Deployment Scenarios](deployment-scenarios.md) | Greenfield vs brownfield migration with phased rollout |
| [Signals Contract](Ai-Grid-Signals-Contract.md) | Cross-cluster signals protocol (Prometheus format, polling, staleness) |

## Diagrams

| Diagram | Preview |
|---------|---------|
| Multi-cluster Data Plane | [View](https://htmlpreview.github.io/?https://github.com/danielezonca/spare-notes/blob/main/grid-mvp/grid-dataplane.html) |
| Tenant Admin Flows | [View](https://htmlpreview.github.io/?https://github.com/danielezonca/spare-notes/blob/main/grid-mvp/grid-tenant-admin.html) |
| Tenant User Flows | [View](https://htmlpreview.github.io/?https://github.com/danielezonca/spare-notes/blob/main/grid-mvp/grid-tenant-user.html) |

## Key Decisions

- **Auth**: API key at Grid level via maas-api (`api_key_auth` filter) — zero user credential change
- **Rate limiting**: MaaS only for MVP (Kuadrant/Limitador), Grid layer deferred
- **External models**: MaaS level for MVP (must stay rate-limited), Grid level post-MVP
- **Model listing**: maas-api reads Grid overlay for cross-cluster model discovery (SWIM gossip)
- **DB**: Shared PostgreSQL across sites, evolve to hub + reconciler for cross-region
- **Clusters**: All equivalent — hub is a DNS designation, not a deployment difference
- **UIs**: Two standalone apps — Admin (RHCE) and User Portal (DevHub/Backstage)
- **Observability**: ACM Multi-cluster Observability Operator → Thanos → Grafana
