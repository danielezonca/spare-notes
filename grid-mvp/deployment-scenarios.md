# AI Grid — Deployment Scenarios

Two distinct paths to multi-cluster, depending on starting point.

---

## Scenario A — Greenfield / Target Architecture

No existing MaaS deployment. Grid is the platform from day one.

### Traffic flow

```
Consumer → MaaS GW (ingress) → Grid GW (routing only) → standalone GW → EPP → vLLM
                         → external API (directly, Grid bypassed)
```

### How each concern is handled

| Concern | How it works |
|---------|-------------|
| **Auth** | API key via maas-api at MaaS GW (Authorino/Kuadrant). Grid GW does routing + service-level auth (SA tokens), no user API key auth. API key management (create, revoke, list) via maas-api remains the user-facing credential system |
| **Rate limiting** | MaaS level (Kuadrant/Limitador) for MVP. Grid `token_rate_limit` with Valkey deferred to post-MVP |
| **External models** | MaaS level — `ExternalModel` CRs with direct endpoints to third-party APIs. Grid bypassed for external API traffic |
| **llm-d models** | Registered as `ExternalModel` CRs pointing to Grid GW endpoint. `ai grid` + `external model` reconcilers automate registration |
| **Observability** | Prometheus metrics + Thanos/Grafana dashboards |
| **Model catalog** | All models (llm-d + external) registered as `ExternalModel` in MaaS. Grid overlay provides cross-cluster discovery via SWIM gossip |
| **Key management** | maas-api provides key CRUD, validation, subscription binding. maas-api runs on ingress clusters only |

Even in greenfield, maas-api is the API key management layer. Building
a separate key management system for Grid would be redundant — maas-api
already handles creation, hashing, validation, subscription binding,
revocation, and expiry. The MaaS Gateway calls Authorino → maas-api on
every request for key validation.

### Characteristics

- MaaS-first architecture — auth and limits enforced before Grid
  routing. Grid handles cross-cluster routing only
- Ingress/inference topology: ingress clusters run MaaS GW + maas-api, inference
  clusters run llm-d + standalone GW + Grid GW only
- Ingress clusters can also host models (superset of an inference cluster)
- No migration concerns — no existing users to break
- Model registration automated via reconcilers

---

## Scenario B — Brownfield: Single MaaS cluster → Multi-cluster Grid

Existing MaaS deployments with users, API keys, subscriptions, and
external models already in production. Grid is added incrementally.

### Key constraints

- API keys MUST remain the same (zero user credential change)
- Rate limiting and metering must keep working throughout
- External models must remain rate-limited at all times

### Phase 0 — Today (single cluster, no Grid)

```
Consumer → MaaS Gateway → external API (ExternalModel CR)
                        → local EPP → vLLM
```

Each site runs a standalone MaaS deployment:
- Praxis AI gateway with direct upstream clusters
- API key auth via maas-api
- Rate limiting via Kuadrant/Limitador (per-user, per-model)
- Observability via Prometheus metrics
- External models via `ExternalModel` CR (rate-limited by Limitador)
- No Grid awareness

### Phase 1 — Ingress & Inference with MaaS-first flow

```
Consumer → MaaS GW (ingress) → Grid GW (routing only) → standalone GW → EPP → vLLM
                         → external API (directly, Grid bypassed)
```

What happens:
- Install Grid Operator + Grid GW on each cluster (ingress and inference)
- MaaS GW + maas-api run on ingress clusters only — customer entry point
- MaaS GW handles all auth (Authorino/Kuadrant → maas-api) and rate
  limiting (Limitador). Grid GW does routing only — service-level auth (SA tokens) instead of `api_key_auth`
- llm-d models attach to standalone GWs, not MaaS GW
- All models (llm-d + external) registered as `ExternalModel` CRs in
  MaaS. llm-d models point to Grid GW endpoint; external APIs point
  to their provider endpoint directly
- `ai grid` reconciler auto-creates `InferenceProvider` CRs from
  opted-in llm-d deployments (annotation + namespace filter)
- `external model` reconciler auto-creates `ExternalModel` +
  `MaaSModelRef` CRs on ingress clusters from InferenceProvider state
- Grid `token_rate_limit` is **disabled** — MaaS/Limitador handles limits
- Shared DB across ingress clusters for API key consistency
- MaaS CRDs (subscriptions, auth policies) replicated via ACM/GitOps
- Inference clusters: standalone GW + llm-d + Grid GW + Grid Operator.
  No MaaS GW, no maas-api. May be air-gapped from public internet
  (private mesh connectivity required)

What doesn't change for users:
- Same API key
- Same models work
- Same rate limits apply
- URL changes only if stable DNS wasn't pre-staged

### Phase 2 — Enhanced integration

- maas-api validation response extended with region + budget metadata
- MaaS GW sets region headers for Grid GW geo fencing via
  `intelligent_route` header-based region matching
- Grid Operators gossip and poll signals across sites (established
  in Phase 1, enhanced with additional signal types)
- MaaS rate limiting continues — still the single enforcement point
- maas-api model listing covers all clusters (via reconciler-created
  MaaSModelRef CRs — no maas-api code change)

### Phase 3 — Full multi-cluster

- All sites enrolled in the Grid mesh
- Consumers enter through any site's Grid Gateway
- Grid makes cost-aware, load-aware, geo-aware routing across all capacity
- Signals polling provides real-time load visibility across sites
- MaaS handles per-model limits, external providers at each site

### Phase 4 — Grid-native external models + rate limiting (optional)

Only when cross-site global budgets become a requirement:

- Enable Grid `token_rate_limit` with Valkey (per-subject, cross-site)
- Create `InferenceProvider` CRs with `backendKind: api_provider` for
  external APIs — Grid routes directly with its own limits and metering
- MaaS rate limiting continues for per-model limits (complementary):
  - Grid: "alice can use 500K tokens/day total across all sites"
  - MaaS: "team-a can use 1M tokens/month on claude-sonnet on this site"
- Retire MaaS-level `ExternalModel` config once Grid-level is validated

---

## Migration Limitations (Scenario B)

### No smooth transition from single-cluster to multi-cluster

Moving from Phase 0 (single-cluster MaaS) to Phase 1 (ingress &
inference) is a **reconfiguration, not an incremental addition**.
The single-cluster MaaS deployment must be restructured:

| Aspect | Phase 0 (single cluster) | Phase 1 (ingress & inference) | Migration action |
|--------|-------------------------|-------------------------------|-----------------|
| Model attachment | Models attached to MaaS GW via `LLMInferenceService` | Models attached to standalone GW, decoupled from MaaS GW | Redeploy models on standalone GW with opt-in annotation |
| Model CRDs | `LLMInferenceService` + `MaaSModelRef` | `ExternalModel` + `MaaSModelRef` (auto-created by reconciler) | Old `LLMInferenceService` refs retired; reconciler generates new CRDs |
| Request path | MaaS GW → EPP → vLLM (direct) | MaaS GW → Grid GW → standalone GW → EPP → vLLM | Install Grid GW + Grid Operator; reconfigure MaaS GW routing |
| External models | `ExternalModel` → MaaS GW routes directly | Same (unchanged for third-party APIs) | None for external models |

This means Phase 0 → Phase 1 requires a planned maintenance window
or a parallel-run migration where both old and new paths are active
during the transition.

### Admin UI model deployment — open question

The existing RHCE Admin UI can deploy standalone
`LLMInferenceService` workloads attached to any Gateway (not only
MaaS GW). This means the UI can already deploy llm-d models on a
standalone Gateway in the ingress & inference architecture.

The open question is whether the MaaS integration works
out-of-the-box once the reconciler chain is in place:

1. **Opt-in annotation**: The `ai grid` reconciler requires an
   annotation (`grid.praxis-proxy.io/managed: "true"`) on the llm-d
   Gateway. Can the existing UI set this annotation, or does it need
   a minor update?

2. **Reconciler chain**: Once the annotation is set, the `ai grid`
   reconciler creates `InferenceProvider` → SWIM propagates →
   `external model` reconciler creates `ExternalModel` +
   `MaaSModelRef`. This should work without UI changes — the UI
   deploys the model, the reconciler chain handles the MaaS
   registration automatically.

3. **MaaS governance**: The platform admin still needs to create
   `MaaSSubscription` + `MaaSAuthPolicy` for the model to become
   `Ready`. This is a separate admin action (via UI or GitOps) and
   is independent of the deployment path.

**Likely outcome**: If the UI can set the opt-in annotation on the
Gateway (or if it's a default annotation configured at the
platform level), the existing deployment flow works with the
reconciler chain — no UI changes needed. Needs verification.

---

## Customer Impact During Migration (Scenario B)

**Key requirement: API keys MUST remain the same.** A tenant user should
not need to regenerate or reconfigure their API key when the platform
moves from single-cluster to multi-cluster. The key was minted by
maas-api, stored as a hash in the DB — as long as the DB is shared
(Phase 1) or replicated (Phase 2+), the same key validates on any site.

### What changes for the user

| Concern | Single cluster (Phase 0) | Multi-cluster (Phase 1+) | Impact |
|---------|-------------------------|-------------------------|--------|
| API key | `sk-oai-abc123...` | Same key | **None** |
| Base URL | `https://ai-gateway.site-a.example.com` | `https://ai-gateway.example.com` (DNS) | **Config change** if no stable DNS |
| Models available | Site-local only | All sites' models | Transparent improvement |
| Rate limits | Per-model (MaaS) | Same | **None** |
| Observability | Site-local Prometheus | Aggregated via Thanos | Transparent improvement |

### URL migration strategy

- **With DNS (recommended)**: Put a stable CNAME (`ai-gateway.example.com`)
  in front from day one, even in single-cluster. When Grid goes live, DNS
  routes to the nearest site's Grid Gateway. Users never change their URL.

- **Without DNS**: Users need to update `ANTHROPIC_BASE_URL` /
  `OPENAI_BASE_URL` once. Minimize by setting up stable DNS before
  migration. Old URL can keep working during a transition period (old
  MaaS gateway stays active, Grid Gateway sits in front).

### Migration checklist for zero-disruption

1. Set up stable DNS name before Grid rollout
2. Verify API keys work on the new endpoint (shared DB)
3. Communicate URL change (if DNS wasn't pre-staged) with transition
   period where both old and new URLs work
4. Old MaaS gateway remains active — becomes the site-local gateway
   behind the Grid Gateway

---

## Infrastructure Decisions (both scenarios)

### Ingress & Inference topology

Two cluster roles, distinguished by MaaS presence:

**Ingress cluster** (per region — customer entry point):
```
Ingress cluster:
  MaaS GW + maas-api + maas-controller   (auth, limits, metering)
  Grid GW + Grid Operator                 (routing, gossip, signals)
  Standalone GW + llm-d (optional)        (inference, if hosting models)
  external model reconciler               (creates ExternalModel CRs)
  ACM agent                               (CRD replication)
  Tenant Admin UI + Management API        (stateless, provided by RHCE)
  Tenant User UI                          (stateless)
```

**Inference cluster** (inference workloads only — can be disconnected
from the public internet; only private network connectivity to other
clusters is required for SWIM gossip, signals polling, and mTLS):
```
Inference cluster:
  Standalone GW + llm-d                   (inference)
  Grid GW + Grid Operator                 (routing, gossip, signals)
  ai grid reconciler                      (creates InferenceProvider CRs)
  ACM agent                               (CRD replication)
```

A single cluster can serve both roles — there is no hard requirement
to separate them. When both roles are on one cluster, it runs the
full stack. The distinction is about MaaS presence, not model
placement.

DNS routes to the nearest ingress cluster:
- `admin.example.com` → Admin UI (any ingress cluster)
- `portal.example.com` → User UI (any ingress cluster)
- `ai-gateway.example.com` → nearest MaaS Gateway (ingress)

**Rationale**:
- MaaS GW + maas-api only where needed (user-facing entry points)
- Spokes can be air-gapped from internet — only private mesh required
- Ingress HA/DR — multiple ingress clusters per region, DNS failover
- UIs are stateless — any ingress cluster can serve them
- Grid routing is symmetric — every site participates in SWIM gossip

### Shared DB: ingress clusters only

**Phase 1 (shared DB):** All ingress clusters connect to the same
PostgreSQL instance. API keys and their hashes are immediately
consistent across ingress clusters. Inference clusters do not connect — they have
no maas-api.

MaaS CRDs (`MaaSSubscription`, `MaaSAuthPolicy`, `AITenant`,
`MaasTenantConfig`) are replicated across ingress clusters via
ACM/GitOps. `ExternalModel` and `MaaSModelRef` CRs are created
independently on each ingress cluster by the `external model` reconciler
(each ingress cluster sees the full mesh state via SWIM gossip).

**Phase 2+ (ingress + reconciler): When ingress clusters span regions or cloud
boundaries, the write path stays on a primary ingress cluster. A reconciler
syncs key hashes to secondary ingress clusters on a multi-second interval.
Key operations are low-frequency — propagation delay is invisible.

### Observability

ACM Multi-cluster Observability Operator:

```
Cluster A (Prometheus) ──┐
                         ├── Thanos (ACM) ── Grafana dashboards
Cluster B (Prometheus) ──┘
```

- Prometheus metrics from all components aggregated in Thanos
- Grafana dashboards: token usage, model latency, cost, rate limit
  utilization, Grid routing decisions
- Admin UI links to Grafana for usage dashboards

### Admin tooling

```
Tenant Admin → Admin UI (RHCE) → Management API (RHCE) → GitOps / ACM / Ansible → clusters
```

- Admin never touches kubectl
- RHCE provides the Admin UI and Management API
- ACM/GitOps distributes MaaS CRDs to target clusters
- MaaS → Grid reconciler auto-creates InferenceProvider CRs

---

## Scenario Comparison

| Concern | Greenfield (A) | Brownfield (B, MVP) |
|---------|---------------|---------------------|
| **Topology** | Ingress/inference from day one | Ingress/inference added to existing single-cluster |
| **Auth** | MaaS GW (Authorino → maas-api) | MaaS GW (Authorino → maas-api) — same |
| **Rate limiting** | MaaS Limitador (MVP); Grid `token_rate_limit` post-MVP | MaaS Limitador only |
| **External APIs** | MaaS `ExternalModel` CR (direct routing, Grid bypassed) | MaaS `ExternalModel` CR (same) |
| **llm-d models** | `ExternalModel` CR → Grid GW → standalone GW | `ExternalModel` CR → Grid GW → standalone GW (same) |
| **Metering** | MaaS-level (Thanos for dashboards) | MaaS-level (Thanos for dashboards) |
| **Model catalog** | All `ExternalModel`. Auto-registered via reconcilers | All `ExternalModel`. Auto-registered via reconcilers |
| **Existing users** | None | Must preserve keys, URLs, limits |
| **Complexity** | Simpler — no migration | More moving parts, phased migration |
| **When to choose** | New platform, no legacy | Existing MaaS users in production |
