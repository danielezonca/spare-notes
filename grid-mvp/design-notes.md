# AI Grid — Design Notes

Tracking architecture decisions, migration strategy, and open questions
for the multi-cluster Grid deployment.

---

## Architecture Overview

The AI Grid is a two-plane system:

- **Control plane**: Grid Operators at each site, connected via SWIM gossip
  (membership, liveness) and signals polling (load metrics). Operators
  reconcile CRDs and generate a routing overlay ConfigMap.
- **Data plane**: Grid Gateways at each site, consuming the overlay to make
  model-based, geo-aware, cost-aware routing decisions. MaaS Gateways
  (Praxis AI) handle site-local model routing, credential injection, and
  forwarding to EPP → vLLM.

### Components per site

| Component | Role |
|-----------|------|
| Grid Gateway | Consumer entry point. API key auth, geo fencing, site selection |
| Grid Operator | SWIM gossip, metrics polling, overlay generation |
| MaaS Gateway (Praxis AI) | Model routing, credential injection, API translation |
| EPP | Endpoint picker — selects a pod within a pool |
| vLLM | Inference backend |

### Inter-site connections

- **Grid Gateways** route cross-site via mTLS
- **Grid Operators** gossip membership/liveness via SWIM (UDP, AES-GCM encrypted)
- **Grid Operators** poll each other's signals via `GET /v1/site/signals` (mTLS, Prometheus format)

---

## Decision: External providers — Grid level vs MaaS level

**Context**: External API providers (Anthropic, OpenAI, OpenRouter) can be
configured at two layers:

1. **Grid level** — `InferenceProvider` CRD with `backendKind: api_provider`.
   Grid-wide visibility, cost-aware scoring, locality score 0.1 (always
   lowest priority fallback). No metrics scraping.
2. **MaaS level** — Praxis AI gateway cluster config with credential
   injection. Site-local only, no Grid visibility or scoring.

**Decision**: MaaS level only for MVP. Grid level deferred to post-MVP.

**Rationale**:
- Rate limiting is at the MaaS level for the MVP (Kuadrant/Limitador).
  All traffic must flow through the MaaS Gateway to be rate-limited.
  If external models were at the Grid level, the Grid Gateway would
  route directly to the external API, bypassing MaaS — **no rate
  limiting, no metering, no credential injection**.
- Existing MaaS deployments already route to external providers at
  the site level via `ExternalModel` CR. This works today.
- The Grid's role in the MVP is cross-site routing between MaaS
  instances only — it never routes directly to a provider.

**MVP traffic flow** (all traffic goes through MaaS):
```
Consumer → Grid Gateway → picks site → MaaS Gateway → external API
                                                     → local EPP → vLLM
```

**Not this** (Grid routes directly, bypasses MaaS enforcement):
```
Consumer → Grid Gateway → external API (no limits, no metering!)
```

**Post-MVP**: When Grid gets its own rate limiting (with Valkey) and
metering integration, external models can optionally move to Grid
level (`InferenceProvider` with `backendKind: api_provider`) for
cost-aware cross-site fallback decisions. At that point the Grid can
enforce limits and meter directly without needing MaaS in the path.

---

## Decision: Model catalog is declarative

**Context**: How does the Grid know which models are available at which site?

**Decision**: Models are statically declared in `InferenceProvider` CRs
(`spec.models`). No auto-discovery from backends.

**Details**:
- Adding a model = update the `InferenceProvider` CR. Operator reconciles
  and regenerates the overlay.
- Metrics scraping collects load signals (queue depth, KV cache), not model
  catalogs.
- No integration with K8s `InferencePool`/`InferenceModel` CRDs from
  gateway-api-inference-extension.
- Site matching uses `siteSelector.matchLabels` against `GridSite` CRs.
- Overlay renderer produces one routing candidate per (model, matched site)
  pair.

**Open question** (resolved): Should automation auto-populate
`InferenceProvider` CRs? Yes — MaaS → Grid reconciler, Phase 2.
See [gaps.md](gaps.md).

### Cross-cluster model discovery via SWIM gossip

The Grid already propagates model information across clusters. The
`GridStateSnapshot` CRDT, exchanged between all sites via SWIM gossip,
contains:

- **`capabilities`** (OrSet) — model names (`Capability::Model("Qwen3-Coder-30B")`),
  plus MCP tools and A2A agents
- **`providers`** (BTreeMap) — per provider: `models[]`, `backend_kind`,
  `phase`, `metrics`, `access_policy`, `capacity_weight`
- **`tenant_spend`** (BTreeMap of GCounter) — per-tenant cumulative spend
  in cents, replicated cross-site

When a new `InferenceProvider` CR is created on Cluster B:
1. Cluster B's Grid Operator builds a `ProviderState` with models, phase, metrics
2. SWIM gossips the `GridStateSnapshot` to all peers (encrypted, bincode)
3. Cluster A's Grid Operator merges via CRDT (commutative, idempotent)
4. Cluster A's overlay now includes `(model, site-B)` candidates
5. Grid Gateway hot-reloads → can route to site-B for that model

**The overlay on each site is a global model catalog**, maintained
automatically by the Grid's gossip protocol. No DB writes, no polling,
no custom sync — the data is already there.

---

## Decision: Multi-cluster model listing via Grid overlay

**Context**: `GET /v1/models` on maas-api uses a K8s informer watching
local `MaaSModelRef` CRs. In multi-cluster, a user hitting Cluster A's
maas-api only sees Cluster A's models — not models deployed on Cluster B.

**Decision**: Extend maas-api's `MaaSModelRefLister` to also read the
Grid overlay ConfigMap (already present as a local file on each cluster
via the `overlay-sync` sidecar).

**How it works**:

```
GET /v1/models → composite lister
  ├── local K8s informer (MaaSModelRef CRs on this cluster)  [existing]
  └── Grid overlay file (models from ALL clusters via SWIM)   [new]
      → merge + deduplicate by model name
      → filter by user's subscriptions (existing logic)
      → return OpenAI-compatible model list
```

The Grid overlay is a local file/ConfigMap that the `overlay-sync`
sidecar already maintains from the Grid Operator's output. maas-api
reads it as a second source — no network calls, no DB, no new
infrastructure. The file contains all models across all sites with
their backend kind, phase, and site information.

**What needs to change**:
- **maas-api**: New `MaaSModelRefLister` implementation that reads
  the Grid overlay file and converts `InferenceProvider` candidates
  to the same model format. Compose with the existing K8s informer
  lister. Deduplicate by model name (same model on multiple sites
  appears once in the listing).
- **Deployment**: Mount the overlay ConfigMap into the maas-api pod
  (same as it's mounted into the Grid Gateway pod).

**Why this approach**:
- No new data path — overlay is already maintained by Grid gossip
- No DB changes — model listing stays informer/file-based
- No network calls — local file read
- The `MaaSModelRefLister` interface is clean and extensible
  (existing interface, new implementation)
- Subscription filtering works unchanged — user sees only models
  their subscription grants access to

**Rejected alternatives**:
- **Shared DB for model metadata**: Net new feature, adds DB writes
  to maas-controller reconciliation, duplicates state (CRDs + DB).
- **Don't solve it (MVP-only)**: Users can call any model by name
  (Grid routes), but can't discover models from other clusters. Poor
  developer experience for a platform serving 8K engineers.
- **Hub-only model listing**: Single point of failure, doesn't work
  if models differ per site.

---

## Deployment Scenarios

See [deployment-scenarios.md](deployment-scenarios.md) for greenfield vs brownfield
migration with phased rollout, customer impact analysis, and
migration checklist.

### Open questions

- **Enrollment automation**: Manual token minting OK for dogfood?
  Or automate for scale?

---

## Management Layer — Personas and Flows

Two personas interact with the management plane. The management layer is
provided by the **models-as-a-service (MaaS)** project — a K8s-native
platform built on OpenShift + Gateway API + Kuadrant (Authorino +
Limitador). It has three components:

- **maas-controller** — K8s operator managing CRDs
- **maas-api** — HTTP API for key management, model listing, subscriptions
- **maas-discovery** — tenant discovery service (scaffold)

### MaaS CRDs (`maas.opendatahub.io/v1alpha1`)

| CRD | Purpose |
|-----|---------|
| `Config` | Cluster singleton. Platform anchor, owns all operands |
| `AITenant` | Bootstraps a tenant: namespace, Gateway ref, OIDC. Supports Praxis via annotation |
| `MaasTenantConfig` | Tenant settings: API key policy, telemetry, scaling |
| `MaaSModelRef` | References a model backend: `LLMInferenceService` (KServe/vLLM) or `ExternalModel` |
| `ExternalModel` | External LLM provider: provider format, targetModel, endpoint, credentialRef |
| `MaaSSubscription` | Grants model access with per-model token rate limits, owner groups, priority, billing metadata |
| `MaaSAuthPolicy` | Authorization policy mapping modelRefs → subjects. Generates Kuadrant AuthPolicies |

**Model readiness rule**: A model is only `Ready` when it has BOTH a
`MaaSSubscription` AND a `MaaSAuthPolicy` targeting it (governance
pairing), plus a healthy backend.

### CRD Relationship Chain

```
Config (cluster)
  └── AITenant (per tenant)
        └── MaasTenantConfig (tenant settings)

MaaSModelRef → references → LLMInferenceService OR ExternalModel
MaaSSubscription → references → MaaSModelRef[] (with tokenRateLimits per model)
MaaSAuthPolicy → references → MaaSModelRef[] + subjects (groups/users)
```

### External Models at MaaS Level

MaaS has first-class support for external providers via `ExternalModel` CR:

```yaml
apiVersion: maas.opendatahub.io/v1alpha1
kind: ExternalModel
metadata:
  name: openai-gpt4o
spec:
  provider: openai
  targetModel: gpt-4o
  endpoint: api.openai.com
  credentialRef:
    name: openai-api-key
```

Referenced by a `MaaSModelRef` with `spec.modelRef.kind: ExternalModel`.
The controller manages networking (HTTPRoute, credential injection via
Authorino).

### Decision: Two standalone UIs, not one

The Tenant Admin UI and Tenant User UI are **separate applications**.

| | Tenant Admin UI | Tenant User UI |
|--|----------------|----------------|
| **Persona** | Platform ops / team lead | Engineer using AI tools |
| **Core job** | Deploy models, configure access, monitor fleet | Get a key, find models, check my usage |
| **Complexity** | High — CRD orchestration, ACM/GitOps, multi-cluster | Low — simple self-service portal |
| **Backend** | Management API → ACM → clusters | maas-api (existing endpoints) |
| **Security** | Elevated access, cluster-aware, RBAC | API key or SSO, no cluster access |
| **Cadence** | Changes rarely, evolves with platform | Changes often, user feedback driven |

**Rationale**:
- Different security posture — admin UI has cluster/ACM access, user
  UI should never touch infra
- Different backends — admin UI orchestrates across clusters via
  ACM/GitOps, user UI just calls maas-api REST endpoints
- Independent evolution — user portal iterates fast on UX, admin UI
  is ops tooling
- User UI can be embedded in Red Hat Developer Hub / Backstage as a
  plugin rather than a standalone app. Impossible if coupled to admin UI

**The boundary between the two is the MaaS data layer** (CRDs + shared
DB). Admin creates models and access rules. User consumes them.

```
Tenant Admin UI → Management API → ACM → clusters → MaaS CRDs
                                                          ↓
                                                    maas-controller reconciles
                                                          ↓
                                               models Ready in overlay + DB
                                                          ↓
Tenant User UI  → maas-api ──────────────── reads models, keys, subscriptions
```

### Tenant Admin

In-cluster operator responsible for a team's AI capacity. Manages CRDs
via kubectl/GitOps and uses admin APIs.

| Action | Mechanism | Status |
|--------|-----------|--------|
| Deploy a self-hosted model | Create `LLMInferenceService` + `MaaSModelRef` | Exists |
| Add an external model | Create `ExternalModel` + `MaaSModelRef` | Exists |
| Grant model access | Create `MaaSSubscription` (rate limits) + `MaaSAuthPolicy` (subjects) | Exists |
| Configure tenant settings | Create/update `MaasTenantConfig` | Exists |
| Bulk revoke API keys | `POST /v1/api-keys/bulk-revoke` | Exists |
| Configure token rate limits | `MaaSSubscription.tokenRateLimits` CRD | Exists |
| Monitor usage | Grafana dashboards (Thanos / ACM observability) | Exists |

**Auth**: K8s RBAC for CRDs.

### Tenant User

External engineer using AI tools (Claude Code, Codex CLI, SDKs). Interacts
only via HTTP APIs — never touches the cluster.

| Action | API | Status |
|--------|-----|--------|
| List available models | `GET /v1/models` on maas-api | Exists |
| List subscriptions | `GET /v1/subscriptions` on maas-api | Exists |
| Create API key | `POST /v1/api-keys` on maas-api (bound to subscription) | Exists |
| List own API keys | `POST /v1/api-keys/search` on maas-api | Exists |
| Revoke own API key | `DELETE /v1/api-keys/:id` on maas-api | Exists |
| Use models | `POST /v1/chat/completions` via gateway | Exists |
| View usage | Grafana dashboards (Thanos / ACM observability) | Exists |

**Auth**: API key or OpenShift token for maas-api. API key for gateway.

### Flow: Model Deployment (Tenant Admin)

```
Tenant Admin
  │
  ├── kubectl apply ExternalModel (or deploy LLMInferenceService)
  ├── kubectl apply MaaSModelRef → references the backend
  ├── kubectl apply MaaSSubscription → token rate limits, owner groups
  └── kubectl apply MaaSAuthPolicy → authorization subjects
      │
      └── maas-controller reconciles
            ├── Creates Kuadrant AuthPolicy + RateLimitPolicy
            ├── Model transitions to Ready (governance paired + backend healthy)
            └── Model appears in GET /v1/models for authorized users
```

### Open questions — Management layer

- **MaaS → Grid auto-reconciliation**: When a tenant admin creates a
  `MaaSModelRef` + `MaaSSubscription`, should a controller automatically
  create the corresponding `InferenceProvider` CR in the Grid? This would
  bridge the MaaS management plane with Grid's routing plane without
  requiring the admin to manage both CRD sets.

- **Where does the management plane live in multi-cluster?**

  **Decision: Option C now, evolve to Option D.**

  **Phase 1 (Option C — shared DB):** All sites run maas-api locally but
  share one RDS instance. Keys, models, and usage are immediately
  consistent. maas-controller runs on the hub only (CRDs are
  cluster-scoped). Simple, no replication logic. Works well when all
  sites are in the same region/cloud.

  **Phase 2 (Option D — hub + cross-cluster reconciler):** When sites
  span regions or cloud boundaries, hub owns the write path (key
  creation, revocation, subscription changes). A reconciler loop syncs
  key hashes and subscription data from hub → peer sites on a
  multi-second interval. Each site's maas-api validates locally against
  its replica. Key operations are low-frequency (create once, use for
  weeks), so a few seconds of propagation delay is invisible — the user
  is still copying the key into their env vars during that window.

  The reconciler could be a simple controller polling the hub DB, or
  PostgreSQL logical replication if staying on RDS. No need for
  sub-second consistency — this is not a trading system.

  Rejected alternatives:
  - **Hub only (Option A)**: Cross-cluster latency on every validation
    call (hot path). Hub is SPOF for all sites.
  - **Per-site isolated (Option B)**: Keys don't work across sites.
    Usage fragmented. Breaks the multi-cluster value proposition.
  - **Grid-native replication (Option E)**: Leverages SWIM/CRDTs but
    is significant new work and mixes concerns between Grid (routing)
    and MaaS (management).

- **Kuadrant vs Grid for rate limiting**: Resolved — see decision below.

---

## Decision: Rate limiting — MaaS only for MVP, Grid later

See [auth-and-ratelimit.md](auth-and-ratelimit.md) for detailed comparison
of MaaS (Kuadrant/Limitador) vs Grid (Praxis token_rate_limit),
MVP decision rationale, and post-MVP migration path.

---

## Decision: Auth migration — API key at the Grid level via maas-api

See [auth-and-ratelimit.md](auth-and-ratelimit.md) for detailed auth flow,
filter chain configuration, maas-api validation response extension,
and AuthenticatedIdentity bridge design.

---

## v3 Demo — What it proves

The v3 experiment (`experiments/llm-d/ai-grid/v3/`) validates three
capabilities in a single Praxis AI binary (no Envoy):

1. **JWT auth** — PPE `user-jwt` plugin, HS256
2. **Geo-residency routing** — `intelligent_route` with `match_claims`
   (JWT `grid_region` claim matched against candidate labels)
3. **Per-subject token budget** — `token_rate_limit`, sliding window

**Limitations of v3**:
- Single cluster, mock backends (not real vLLM)
- Overlay disabled — candidates hardcoded in praxis.yaml
- No Grid Operator, no SWIM, no signals polling
- No MaaS gateway layer (Grid Gateway routes directly to backends)

---

## CRD Reference

### InferenceProvider `backendKind` values

| Value | Locality score | Use case |
|-------|---------------|----------|
| `local` | 1.0 | Self-hosted vLLM on same cluster |
| `remote` | 0.4–0.7 | vLLM on a peer cluster (region-aware) |
| `cloud_managed` | 0.2 | AWS Bedrock, GCP Vertex |
| `api_provider` | 0.1 | OpenAI, Anthropic, OpenRouter |

### Scoring strategies

| Strategy | When used | Signals |
|----------|-----------|---------|
| `noMetrics` | External APIs, default | Locality + cost only |
| `queueDepth` | llm-d backends | Shortest normalized queue |
| `kvCachePressure` | llm-d backends | Most available KV cache |

Six weighted signals available: cost, kv_cache, latency, locality,
prefix_cache, queue_depth.
