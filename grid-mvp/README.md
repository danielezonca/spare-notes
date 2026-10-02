# AI Grid MVP

Architecture design for multi-cluster AI Grid deployment — routing
inference across clusters with MaaS (Models-as-a-Service) integration.

## Documents

| Document | Description |
|----------|------------|
| [Personas](personas.md) | Three personas: Platform Admin, Tenant Admin, Tenant User — roles, boundaries, cluster visibility |
| [Design Notes](design-notes.md) | Architecture decisions, migration strategy, component overview |
| [Gaps](gaps.md) | Changes required to existing components (maas-api, Praxis AI, Grid Operator, infra) |
| [Auth & Rate Limiting](auth-and-ratelimit.md) | Detailed auth and rate limiting flows across Grid and MaaS layers |
| [Deployment Scenarios](deployment-scenarios.md) | Greenfield vs brownfield migration with phased rollout |
| [Signals Contract](Ai-Grid-Signals-Contract.md) | Cross-cluster signals protocol (Prometheus format, polling, staleness) |
| [Ingress & Inference Data Plane](ingress-inference-dataplane.md) | Ingress + inference cluster topology — goal matrix, data plane mapping, adversarial analysis, component breakdown |

## Diagrams

### Animated flow visualizers (FlowStory)

Interactive step-by-step diagrams with animated request flows,
clickable component tooltips, and request/response inspector panels.
Built using the [FlowStory](https://noyitz.github.io/flowstory/)
framework — each diagram is a self-contained HTML file with no
external dependencies, generated from the architecture docs in this
repo using Claude Code as the authoring tool.

| Diagram | Flows | Preview |
|---------|-------|---------|
| Multi-cluster Data Plane | Local routing, Cross-site routing, External model, Auth failure | [View](https://htmlpreview.github.io/?https://github.com/danielezonca/spare-notes/blob/main/grid-mvp/grid-dataplane-flow.html) |
| Platform Admin Flows | Deploy model, Grant model access, Metric scraping, View dashboards | [View](https://htmlpreview.github.io/?https://github.com/danielezonca/spare-notes/blob/main/grid-mvp/grid-platform-admin-flow.html) |
| Tenant Admin Flows | Add user to group, Upgrade user, Remove user access | [View](https://htmlpreview.github.io/?https://github.com/danielezonca/spare-notes/blob/main/grid-mvp/grid-tenant-admin-flow.html) |
| Tenant User Flows | List models, Create API key, Model discovery (SWIM), Use model | [View](https://htmlpreview.github.io/?https://github.com/danielezonca/spare-notes/blob/main/grid-mvp/grid-tenant-user-flow.html) |

### Static diagrams

| Diagram | Preview |
|---------|---------|
| Multi-cluster Data Plane | [View](https://htmlpreview.github.io/?https://github.com/danielezonca/spare-notes/blob/main/grid-mvp/grid-dataplane.html) |
| Platform Admin Flows | [View](https://htmlpreview.github.io/?https://github.com/danielezonca/spare-notes/blob/main/grid-mvp/grid-platform-admin.html) |
| Tenant Admin Flows | [View](https://htmlpreview.github.io/?https://github.com/danielezonca/spare-notes/blob/main/grid-mvp/grid-tenant-admin.html) |
| Tenant User Flows | [View](https://htmlpreview.github.io/?https://github.com/danielezonca/spare-notes/blob/main/grid-mvp/grid-tenant-user.html) |

### Related

The animated diagrams reference the [AI Inference Gateway — MaaS Flow
Visualizer](https://noyitz.github.io/ai-gateway-docs/ai-gateway-flow.html)
for detailed single-cluster MaaS request processing (Envoy → Kuadrant
→ Praxis plugins → llm-d/external providers).

## Key Decisions

- **Topology**: Ingress clusters (MaaS GW + maas-api + Grid GW) / Inference clusters (standalone GW + llm-d + Grid GW — can be disconnected from internet, only inter-cluster connectivity required). A single cluster can serve both roles. See [Ingress & Inference Data Plane](ingress-inference-dataplane.md)
- **Request flow**: MaaS first — Customer → MaaS GW (auth, limits, metering) → Grid GW (routing only) → target GW → backend
- **Auth**: API key at MaaS level via maas-api (Authorino) — zero user credential change. Grid GW does routing + service-level auth (SA tokens), no user API key auth
- **Rate limiting**: MaaS only for MVP (Kuadrant/Limitador), Grid layer deferred
- **Model abstraction**: All models (llm-d + external) registered as `ExternalModel` in MaaS. One CR per distinct model. Automated via `external model` reconciler in Grid
- **llm-d discovery**: Opt-in annotation + namespace filter. `ai grid` reconciler creates `InferenceProvider` CRs
- **External APIs**: MaaS routes directly (Grid bypassed) — existing behavior preserved
- **Model listing**: `external model` reconciler creates MaaSModelRef CRs on each ingress cluster; maas-api's existing informer sees all models (minor changes acceptable)
- **DB**: Shared PostgreSQL across ingress clusters for API keys
- **CRD replication**: ACM/GitOps replicates MaaS CRDs (subscriptions, auth policies, tenant config) across ingress clusters
- **UIs**: RHOAI Customer Portal (standalone RHOAI Dashboard) for tenant users, Admin Console (RHCE) for platform admins
- **Observability**: ACM Multi-cluster Observability Operator → Thanos → Grafana
