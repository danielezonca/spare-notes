# AI Grid — Component Gaps

Changes required to existing components for multi-cluster Grid deployment.
Extracted from design-notes.md decisions.

---

## maas-api

| Gap | Description | Phase | Effort |
|-----|------------|-------|--------|
| **Extend validation response** | `POST /internal/v1/api-keys/validate` must return `region` (from AITenant/MaasTenantConfig) and `token_budget` (from MaaSSubscription.tokenRateLimits) alongside existing username/groups. Grid Gateway needs these for geo fencing and budget enforcement. | Phase 2 | Medium |
| **Composite model lister** | New `MaaSModelRefLister` implementation that reads the Grid overlay file (local ConfigMap maintained by overlay-sync sidecar) and merges with the existing K8s informer. Deduplicates by model name. Returns models from all Grid sites, not just local cluster. | Phase 1 | Medium |
| **Overlay ConfigMap mount** | maas-api pod needs the Grid overlay ConfigMap mounted (same volume mount the Grid Gateway already uses). Deployment/Helm change only. | Phase 1 | Small |
| **Region in tenant config** | `AITenant` or `MaasTenantConfig` CRD needs a `region` field (or equivalent). maas-api reads this during validation to return the tenant's geo region. Requires CRD schema change + controller update. | Phase 2 | Medium |

## Praxis AI (ai repo)

| Gap | Description | Phase | Effort |
|-----|------------|-------|--------|
| **api_key_auth → AuthenticatedIdentity bridge** | The `api_key_auth` filter must populate the `AuthenticatedIdentity` extension (subject_id, roles, custom_claims) from maas-api's validation response. Today the `policy` filter (PPE) does this from JWT claims. Need the same for API keys so downstream filters (`intelligent_route`, `token_rate_limit`) work. | Phase 1 | Medium |
| **intelligent_route: header-based region** | `intelligent_route` currently reads `grid_region` from JWT claims via `match_claims`. Needs to also accept a header-based region source (e.g., `X-Grid-Region` set by `api_key_auth`). Alternative: a new filter that maps validation response fields → headers before `intelligent_route`. | Phase 2 | Medium |
| **Grid Gateway filter chain config** | Define the combined filter chain for the Grid Gateway role: `api_key_auth → model_to_header → intelligent_route → load_balancer` (no `token_rate_limit` for MVP). Needs a documented config template / Helm values. | Phase 1 | Small |
| **Kuadrant + Praxis integration** | MaaS currently uses Kuadrant (Authorino + Limitador) with Envoy/Istio. With Praxis as the MaaS gateway engine (AITenant annotation `payload-processing-type=praxis`), verify Kuadrant integration works: Authorino for auth policies, Limitador for token rate limits. May need adapter or configuration work. | Phase 1 | Medium |

## Grid Operator (grid repo)

| Gap | Description | Phase | Effort |
|-----|------------|-------|--------|
| **MaaS → Grid reconciler** | New controller (could live in grid repo or as a standalone operator) that watches `MaaSModelRef` CRs and auto-creates corresponding `InferenceProvider` CRs. Maps MaaSModelRef fields (model name, backend kind, endpoint) to InferenceProvider spec. Ensures models deployed via MaaS become grid-routable without manual Grid CRD management. | Phase 2 | Large |

## maas-api — Management Layer Auth

| Gap | Description | Phase | Effort |
|-----|------------|-------|--------|
| **Tenant user auth without OpenShift** | Today maas-api auth supports OpenShift tokens (TokenReview) and API keys. Tenant admins have OpenShift accounts — fine. But tenant users are external engineers (8K+) who may NOT have OpenShift accounts. They need to access self-service endpoints (GET /v1/models, POST /v1/api-keys, GET /v1/subscriptions) without requiring an OpenShift login. Options: (a) API key auth for management endpoints (key authenticates itself), (b) SSO/OIDC integration (corporate IdP, not OpenShift), (c) ephemeral token from portal login. Need to design the auth flow for the Tenant User UI → maas-api path. | Phase 1 | Medium |

## MaaS Controller (maas-controller)

| Gap | Description | Phase | Effort |
|-----|------------|-------|--------|
| **Region field in CRD** | `AITenant` or `MaasTenantConfig` needs `spec.region` (or `spec.gridRegion`). Controller must propagate this to maas-api's runtime config so validation responses include it. | Phase 2 | Small |

## Deployment / Helm / Infrastructure

| Gap | Description | Phase | Effort |
|-----|------------|-------|--------|
| **Shared DB setup** | All clusters must connect to the same PostgreSQL instance. Connection string in maas-api config. Requires network connectivity (VPC peering, private endpoints) between clusters and DB. | Phase 1 | Medium |
| **Grid Operator installation** | Install Grid Operator + Grid Gateway on each cluster via Helm charts (already exist in grid repo: `grid-operator`, `grid-site`, `praxis-gateway`). SWIM seed configuration for the hub. | Phase 1 | Medium |
| **Site enrollment** | Each site needs a one-time enrollment token to join the Grid mesh. Manual for dogfood, automate for scale. | Phase 1 | Small |
| **overlay-sync sidecar** | Deploy overlay-sync sidecar alongside maas-api (not just Grid Gateway) so it can read the overlay for model listing. | Phase 1 | Small |
| **ACM / GitOps pipeline** | Configure ACM to distribute MaaS CRDs (`MaaSModelRef`, `ExternalModel`, `MaaSSubscription`, `MaaSAuthPolicy`) and Grid CRDs to target clusters. This connects management-plane model declarations to the multi-cluster deployment. | Phase 1 | Medium |
| **Thanos / Grafana** | Configure ACM Multi-cluster Observability to aggregate Prometheus metrics from all clusters. Provide dashboards for Grid routing decisions, token usage, latency, capacity, and cost across the integrated deployment. | Phase 1 | Medium |

## Future Grid-dependent product requirements

These are not changes required for the Grid MVP. They may require an
explicit Grid-to-enforcement or Grid-to-routing contract in a later
phase, so they are retained here only as cross-references. The broader
quota, metering, billing, key-management, and MaaS enforcement backlog
is tracked in [`../tokenomics/research/adversarial-review.md`](../tokenomics/research/adversarial-review.md)
and [`../pricetag/`](../pricetag/).

| ID | Category | Requirement | Grid dependency | Phase |
|----|----------|-------------|-----------------|-------|
| G-01 | quota-management | **Capacity-informed quotas** — derive quota configuration from available GPU capacity, replica topology, throughput, and latency bounds. | Grid capacity signals could become an input to a separate quota-control loop; they do not configure quotas in the MVP. | Post-MVP |
| G-02 | routing | **Quota-exhaustion fallback routing** — select a lower-cost or on-prem fallback rather than returning a quota rejection. | Requires an enforcement-to-routing contract and an explicit fallback policy. The MVP enforces quotas in MaaS and has no such contract. | Post-MVP |

---

## Summary by Phase

### Phase 1 (Grid alongside MaaS)
- Grid Operator + Gateway installation on all clusters
- Site enrollment
- Shared DB connectivity
- ACM/GitOps pipeline for MaaS and Grid CRD distribution
- maas-api: overlay mount + composite model lister
- Praxis AI: Grid Gateway filter chain config + AuthenticatedIdentity bridge
- overlay-sync sidecar for maas-api
- Thanos/Grafana observability across the integrated deployment
- Tenant user auth without OpenShift (management layer)
- Kuadrant + Praxis integration verification

### Phase 2 (enhanced integration)
- maas-api: extended validation response (region, budget)
- MaaS CRD: region field in AITenant/MaasTenantConfig
- Praxis AI: header-based region in intelligent_route
- MaaS → Grid auto-reconciler

### Post-MVP
- Grid `token_rate_limit` with Valkey
- Grid-level external models (`InferenceProvider` api_provider)
- Grid-level metering integration
- Capacity-informed quotas (requires a Grid-to-enforcement contract)
- Quota-exhaustion fallback routing (requires an enforcement-to-routing contract)
