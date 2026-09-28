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
| **Stable DNS** | CNAME `ai-gateway.example.com` pointing to the hub cluster's ingress. Set up before Grid rollout to avoid user URL changes. Also `admin.example.com` and `portal.example.com` for UIs. | Phase 0 | Small |
| **ACM / GitOps pipeline** | Configure ACM to distribute MaaS CRDs (MaaSModelRef, ExternalModel, MaaSSubscription, MaaSAuthPolicy) to target clusters. | Phase 1 | Medium |
| **Thanos / Grafana** | ACM Multi-cluster Observability Operator configured to aggregate Prometheus metrics from all clusters. Grafana dashboards for token usage, latency, cost. | Phase 1 | Medium |

## Gaps from Adversarial Review

Requirements found in customer and Pricetag sources that were not
previously captured. See `tokenomics/research/adversarial-review.md`
for the full cross-reference analysis.

### Critical — Customer expects these

| Gap | Description | Phase | Effort |
|-----|------------|-------|--------|
| **Concurrent request limits** | Max N simultaneous requests per app+model. Customer currently enforces this (TPM, TPH, concurrent caps). Neither `token_rate_limit`, Limitador, nor `external_metering` supports concurrency limiting. Needs a semaphore-style filter or connection limit at LB level. | Post-MVP | Large |
| **Hierarchical quotas** | Cascading limits: org → team → user. Each level enforced independently — user within personal quota can still be blocked by exhausted org bucket. Praxis Epic #121 mentions this. Depends on compositional keys (#1334) + simultaneous multi-scope limits (#979). | Post-MVP | Large |
| **Error refund behavior** | When provider errors after token reservation, are reserved tokens refunded? Customer expects full refund on system errors. Praxis reserve/reconcile exists but error path not documented or verified. | Phase 1 | Small (verify + document) |
| **429 response headers (full set)** | Customer expects `Retry-After`, `X-RateLimit-Limit-Tokens`, `X-RateLimit-Remaining-Tokens`, `X-RateLimit-Reset`. Praxis Epic #121 specifies these. Verify `token_rate_limit` emits the full set. | Phase 1 | Small (verify) |
| **Quota warning thresholds** | Alert users via response header at 80% and 90% of budget. Prevents surprise 429s. Could be done via `token_rate_limit` response enrichment or `external_metering` response headers. | Post-MVP | Medium |
| **Per-app vs per-user identity** | Application identity (project ID) is distinct from user identity. Rate limits can be keyed on either independently. MaaS identity model doesn't distinguish app from user. Praxis compositional keys (#1334) could key on app header, but MaaS identity model needs extension. | Post-MVP | Large |
| **GPU capacity-based quota calculation** | Rate limits derived from actual GPU capacity (replica counts, tensor parallelism, throughput bounds). Grid scoring has capacity signals but doesn't feed into quota configuration. | Post-MVP | Medium |

### Important — Needed for production

| Gap | Description | Phase | Effort |
|-----|------------|-------|--------|
| **Token type preservation in migration** | Migration from `external_metering` to `token_rate_limit` must preserve per-type tracking: cache_writes, cache_reads, prompt, completion, reasoning. Metering CloudEvents and billing need these broken out. Praxis has token-type weights (#1132) but the recording side needs the same granularity. | Phase 1 (migration) | Medium |
| **Key rotation procedure** | Documented workflow for rotating provider API keys and user API keys (create new → migrate clients → revoke old). maas-api has revocation but no rotation workflow. | Phase 1 | Small |
| **Service account keys for CI/CD** | Key type with configurable or no quota for CI/CD pipelines. Not counted against user budgets. MaaS has ephemeral keys but not a "service account" type. | Phase 2 | Medium |
| **GLM free tier fallback** | When frontier model quota is exhausted, route to free on-premise model instead of 429. Requires soft enforcement (#1381) + intelligent routing to redirect to free tier. | Post-MVP | Medium |
| **Cross-provider cost visualization** | Unified view of token usage and cost across on-prem GPU + external API billing (Anthropic, OpenAI, Vertex). Thanos handles metrics but not billing reconciliation across providers. | Post-MVP | Large |
| **Cost center mapping for chargeback** | Map user → cost center (from LDAP/HR data) for internal cross-charging. Requires integration with corporate identity systems. | Post-MVP | Medium |
| **Onboarding guardrails (max output tokens)** | Admin-configured max output token ceiling during team onboarding. Prevents unrealistic max_tokens values that starve shared GPU capacity. Pre-forward token ceiling (#1332) handles runtime; onboarding needs admin config. | Phase 2 | Small |
| **Rate limiting scope mismatch** | Customer uses per-APP rate limiting (app+model). MaaS Limitador uses per-SUBSCRIPTION (subscription+model). These are not the same. Need to clarify whether MaaS subscription ≈ app, or if a separate app identity is needed. | Phase 1 (clarify) | Small (decision) |

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
- Verify: error refund behavior in reserve/reconcile
- Verify: 429 response headers (full set)
- Clarify: rate limiting scope — MaaS subscription vs customer app identity
- Key rotation procedure
- Token type preservation plan for migration

### Phase 2 (enhanced integration)
- maas-api: extended validation response (region, budget)
- MaaS CRD: region field in AITenant/MaasTenantConfig
- Praxis AI: header-based region in intelligent_route
- MaaS → Grid auto-reconciler
- Service account keys for CI/CD
- Onboarding guardrails (admin-configured max output tokens)

### Post-MVP
- Grid `token_rate_limit` with Valkey
- Grid-level external models (`InferenceProvider` api_provider)
- Grid-level metering integration
- Concurrent request limits (semaphore filter)
- Hierarchical quotas (org → team → user)
- Quota warning thresholds (80%/90% headers)
- Per-app vs per-user identity distinction
- GPU capacity-based quota calculation
- GLM free tier fallback routing
- Cross-provider cost visualization
- Cost center mapping for chargeback
