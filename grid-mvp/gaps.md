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

> **Note**: Per-user quota and usage endpoints (`PATCH /admin/quotas`,
> `GET /admin/usage`, `GET /v1/usage/me`, `GET /v1/quota/me`) are
> Pricetag dogfood concerns (metering-service), not Grid scope.
> In the Grid architecture, quota is managed via `MaaSSubscription.tokenRateLimits`
> (CRD-driven) and usage dashboards via Thanos/Grafana (ACM observability).

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
| **Stable DNS** | CNAME `ai-gateway.example.com` pointing to the hub cluster's ingress. Set up before Grid rollout to avoid user URL changes. Also `admin.example.com` and `portal.example.com` for UIs. | Phase 0 | Small |
| **ACM / GitOps pipeline** | Configure ACM to distribute MaaS CRDs (MaaSModelRef, ExternalModel, MaaSSubscription, MaaSAuthPolicy) to target clusters. | Phase 1 | Medium |
| **Thanos / Grafana** | ACM Multi-cluster Observability Operator configured to aggregate Prometheus metrics from all clusters. Grafana dashboards for token usage, latency, cost. | Phase 1 | Medium |

---

## Summary by Phase

### Phase 0 (pre-Grid)
- Stable DNS setup

### Phase 1 (Grid alongside MaaS)
- Grid Operator + Gateway installation on all clusters
- Site enrollment
- Shared DB connectivity
- ACM/GitOps pipeline for CRD distribution
- maas-api: overlay mount + composite model lister
- Praxis AI: Grid Gateway filter chain config + AuthenticatedIdentity bridge
- overlay-sync sidecar for maas-api
- Thanos/Grafana observability
- Tenant user auth without OpenShift (management layer)
- Kuadrant + Praxis integration verification

### Phase 2 (enhanced integration)
- maas-api: extended validation response (region, budget)
- MaaS CRD: region field in AITenant/MaasTenantConfig
- Praxis AI: header-based region in intelligent_route
- MaaS → Grid auto-reconciler

### Phase 4 (optional, post-MVP)
- Grid `token_rate_limit` with Valkey
- Grid-level external models (`InferenceProvider` api_provider)
- Grid-level metering integration
