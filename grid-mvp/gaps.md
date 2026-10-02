# AI Grid — Component Gaps

Changes required to existing components for multi-cluster Grid deployment.
Extracted from design-notes.md decisions.

---

## maas-api

**Approach**: Prefer existing maas-api APIs and CRD-based data flow.
Minor code changes are acceptable when they simplify integration
logic compared to working around the existing API surface.

The `external model` reconciler creates `MaaSModelRef` CRs on each
ingress cluster for every model in the Grid mesh. maas-api's existing
K8s informer watches `MaaSModelRef` CRs locally, so `GET /v1/models`
returns all models (local + remote) without changes. API key
validation uses the existing `/internal/v1/api-keys/validate`
endpoint.

| Gap | Description | Phase | Effort |
|-----|------------|-------|--------|
| **Geo fencing metadata** | When geo fencing is needed, MaaS GW must obtain the tenant's region. Options: (a) extend maas-api validation response to include `region` from `AITenant.spec.region` (simplest — small code change), (b) Authorino reads the region from CRD metadata directly. Evaluate during Phase 2 based on which path is simpler. | Phase 2 | Medium |

## Praxis AI (ai repo)

| Gap | Description | Phase | Effort |
|-----|------------|-------|--------|
| **intelligent_route: header-based region** | `intelligent_route` currently reads `grid_region` from JWT claims via `match_claims`. Needs to also accept a header-based region source (e.g., `X-Grid-Region` set by MaaS GW from the maas-api validation response). | Phase 2 | Medium |
| **Grid Gateway filter chain config** | Define the filter chain for the Grid Gateway routing-only role: `model_to_header → intelligent_route → load_balancer`. No `api_key_auth` (MaaS handles auth). No `token_rate_limit` for MVP. Needs a documented config template / Helm values. | Phase 1 | Small |
| **Kuadrant + Praxis integration** | MaaS currently uses Kuadrant (Authorino + Limitador) with Envoy/Istio. With Praxis as the MaaS gateway engine (AITenant annotation `payload-processing-type=praxis`), verify Kuadrant integration works: Authorino for auth policies, Limitador for token rate limits. May need adapter or configuration work. | Phase 1 | Medium |

## Grid Operator (grid repo)

| Gap | Description | Phase | Effort |
|-----|------------|-------|--------|
| **`ai grid` reconciler** | New controller in Grid Operator (Rust). Watches llm-d Gateway CRDs with opt-in annotation (`grid.praxis-proxy.io/managed: "true"`) in namespaces matching a configurable filter. Creates `InferenceProvider` CRs with `backendKind: "local"`, mapping model names and standalone GW endpoint from the llm-d deployment. Updates on spec changes; deletes when annotation is removed or Gateway is deleted. Runs on every cluster hosting llm-d models (ingress or inference). | Phase 1 | Large |
| **`external model` reconciler** | New controller in Grid Operator (Rust). Watches `InferenceProvider` CRs (which reflect the full mesh state via SWIM gossip). Creates `ExternalModel` + `MaaSModelRef` CRs in the MaaS API group (`maas.opendatahub.io/v1alpha1`). One `ExternalModel` per distinct model name — deduplicates across providers on multiple sites. ExternalModel endpoint = local Grid GW K8s service FQDN. Does NOT create `MaaSSubscription` or `MaaSAuthPolicy` (admin concerns, managed via ACM/GitOps). Runs on ingress clusters only (guarded by feature flag or CRD presence check). | Phase 1 | Large |
| **Namespace filter for llm-d discovery** | New field on `GridNetwork` spec (or new CRD) to configure which namespaces and annotation key/value the `ai grid` reconciler watches. E.g., `llmdDiscovery.namespaces`, `llmdDiscovery.annotation`. | Phase 1 | Small |

## Management Layer Auth

| Gap | Description | Phase | Effort |
|-----|------------|-------|--------|
| **Tenant user auth without OpenShift** | Tenant users are external engineers (8K+) who may not have OpenShift accounts. They need to access maas-api self-service endpoints (GET /v1/models, POST /v1/api-keys, GET /v1/subscriptions) without OpenShift login. Options: (a) add SSO/OIDC token validation to maas-api (minor code change — accept corporate IdP tokens alongside existing OpenShift tokens), (b) OAuth2 proxy in front of maas-api that translates OIDC tokens to OpenShift-compatible headers (infrastructure-only), (c) API key auth for management endpoints (maas-api already supports this — key authenticates itself). | Phase 1 | Medium |

## MaaS Controller (maas-controller)

| Gap | Description | Phase | Effort |
|-----|------------|-------|--------|
| **Region field in CRD** | `AITenant` or `MaasTenantConfig` needs `spec.region` (or `spec.gridRegion`). Used by Authorino (via Kuadrant metadata source) for geo fencing decisions at the MaaS GW level. CRD schema change + controller update to propagate the field. | Phase 2 | Small |

## Deployment / Helm / Infrastructure

| Gap | Description | Phase | Effort |
|-----|------------|-------|--------|
| **Ingress/Inference topology deployment** | Define cluster roles via Helm values, cluster labels, or ACM placement rules. Ingress clusters run MaaS GW + maas-api + maas-controller + `external model` reconciler. Inference clusters run Grid GW + standalone GW + llm-d + `ai grid` reconciler only. A single cluster can serve both roles — there is no hard requirement to separate them. When a cluster is both ingress and inference, it runs the full stack (MaaS GW + Grid GW + standalone GW + llm-d + both reconcilers). | Phase 1 | Medium |
| **Standalone GW deployment** | Each cluster hosting llm-d models (ingress or inference) needs a standalone Gateway separate from MaaS GW and Grid GW. llm-d models attach to this gateway, not to MaaS GW. Helm chart or kustomize overlay for the standalone GW. | Phase 1 | Medium |
| **Inter-service auth (MaaS GW → Grid GW → llm-d GW)** | Authentication between the gateway layers. MaaS GW → Grid GW and Grid GW → standalone llm-d GW must authenticate requests — not with user API keys, but with service-level credentials. Options: (a) Kubernetes ServiceAccount tokens (projected SA token validated by the receiving gateway), (b) mTLS with service mesh certificates, (c) static shared secret. SA tokens (option a) are the recommended approach: no manual secret management, integrates with K8s RBAC, and the ExternalModel `credentialRef` can reference the projected token. Cross-cluster Grid GW → Grid GW traffic is already authenticated via Grid mTLS. | Phase 1 | Medium |
| **Shared DB setup** | Ingress clusters must connect to the same PostgreSQL instance (inference clusters do not — they have no maas-api). Connection string in maas-api config. Requires network connectivity (VPC peering, private endpoints) between ingress clusters and DB. | Phase 1 | Medium |
| **Grid Operator installation** | Install Grid Operator + Grid Gateway on each cluster via Helm charts (already exist in grid repo: `grid-operator`, `grid-site`, `praxis-gateway`). SWIM seed configuration for the ingress cluster. | Phase 1 | Medium |
| **Site enrollment** | Each site needs a one-time enrollment token to join the Grid mesh. Manual for dogfood, automate for scale. | Phase 1 | Small |
| **ACM / GitOps pipeline** | Configure ACM to distribute admin-managed MaaS CRDs (`MaaSSubscription`, `MaaSAuthPolicy`, `AITenant`, `MaasTenantConfig`) across ingress clusters. `ExternalModel` and `MaaSModelRef` CRs are NOT distributed by ACM — they are auto-created independently on each ingress cluster by the `external model` reconciler (each ingress cluster sees the full mesh via SWIM gossip). Grid CRDs (`InferenceProvider`) are auto-created by the `ai grid` reconciler on each cluster, not distributed by ACM. | Phase 1 | Medium |
| **Thanos / Grafana** | Configure ACM Multi-cluster Observability to aggregate Prometheus metrics from all clusters. Provide dashboards for Grid routing decisions, token usage, latency, capacity, and cost across the integrated deployment. | Phase 1 | Medium |
| **Single-cluster → multi-cluster migration plan** | No smooth transition from single-cluster MaaS to ingress & inference topology. Models must be redeployed on standalone GWs (currently attached to MaaS GW via `LLMInferenceService`). MaaS GW routing must be reconfigured. Requires a planned maintenance window or parallel-run migration. See [deployment-scenarios.md](deployment-scenarios.md). | Phase 1 | Medium |
| **Admin UI: opt-in annotation** | Existing RHCE Admin UI can deploy standalone `LLMInferenceService` on any Gateway. Verify whether the UI can set the opt-in annotation (`grid.praxis-proxy.io/managed: "true"`) on the llm-d Gateway. If yes, the existing deployment flow works with the reconciler chain — no UI changes needed. If not, a minor UI update or a platform-level default annotation is needed. See [deployment-scenarios.md](deployment-scenarios.md). | Phase 1 | Small |

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

### Phase 1 (Ingress & Inference deployment)
- **Topology**: Define cluster roles (Helm values, ACM placement). A single cluster can be both ingress and inference — no hard separation required
- **Ingress clusters**: MaaS GW + maas-api + Grid GW (routing only) + Grid Operator
- **Inference clusters**: standalone GW + llm-d + Grid GW + Grid Operator (can be disconnected from internet — only inter-cluster connectivity required)
- **`ai grid` reconciler**: llm-d → InferenceProvider (Grid Operator, Rust)
- **`external model` reconciler**: InferenceProvider → ExternalModel + MaaSModelRef on ingress clusters (Grid Operator, Rust)
- **Namespace filter**: GridNetwork spec extension for llm-d discovery
- **Standalone GW deployment**: Helm chart / kustomize for llm-d-attached gateway
- **Inter-service auth**: SA tokens for MaaS GW → Grid GW and Grid GW → llm-d GW; Grid mTLS for cross-cluster
- Grid Operator + Grid GW installation on all clusters
- Site enrollment
- Shared DB connectivity (ingress clusters only)
- ACM/GitOps pipeline for admin-managed MaaS CRD distribution across ingress clusters
- maas-api: prefer existing APIs; minor changes acceptable to simplify integration
- Praxis AI: Grid GW filter chain (`model_to_header → intelligent_route → load_balancer`)
- Thanos/Grafana observability across the integrated deployment
- Tenant user auth without OpenShift
- Kuadrant + Praxis integration verification

### Phase 2 (enhanced integration)
- MaaS CRD: region field in AITenant/MaasTenantConfig
- Geo fencing: Authorino reads tenant region from CRD, sets header for Grid GW intelligent_route
- Praxis AI: header-based region in intelligent_route (source: MaaS GW headers)

### Post-MVP
- Grid `token_rate_limit` with Valkey
- Grid-level external models (`InferenceProvider` api_provider)
- Grid-level metering integration
- Capacity-informed quotas (requires a Grid-to-enforcement contract)
- Quota-exhaustion fallback routing (requires an enforcement-to-routing contract)
