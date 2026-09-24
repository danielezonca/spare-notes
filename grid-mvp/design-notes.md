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
| Grid Gateway | Consumer entry point. JWT auth, geo fencing, token budget, site selection |
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

**Open question**: Should automation (e.g., a controller watching vLLM
deployments) auto-populate `InferenceProvider` CRs? Currently manual.

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

Two distinct paths to multi-cluster, depending on starting point.

### Scenario A — Greenfield / Target Architecture

No existing MaaS deployment. Grid is the platform from day one.

```
Consumer → Grid Gateway → picks site → site gateway → EPP → vLLM
                       → external API (api_provider)
```

| Concern | How it works |
|---------|-------------|
| **Auth** | API key via maas-api (`api_key_auth` filter). Optionally also JWT from IdP for service-to-service. API key management (create, revoke, list) via maas-api remains the user-facing credential system |
| **Rate limiting** | Grid `token_rate_limit` with Valkey (per-subject, cross-site global) |
| **External models** | Grid `InferenceProvider` with `backendKind: api_provider`. Grid routes directly, enforces limits and meters |
| **Metering** | Grid-level token counting and metering callout |
| **Model catalog** | `InferenceProvider` CRs declare models. Overlay renders candidates |
| **Key management** | maas-api provides key CRUD, validation, subscription binding. This is not a MaaS-vs-Grid concern — API key management is a platform service used by both scenarios |

Even in greenfield, maas-api is the API key management layer. Building
a separate key management system for Grid would be redundant — maas-api
already handles creation, hashing, validation, subscription binding,
revocation, and expiry. The Grid Gateway calls `api_key_auth` →
maas-api on every request, same as the brownfield path.

### Scenario B — Brownfield: Single MaaS cluster → Multi-cluster Grid

Existing MaaS deployments with users, API keys, subscriptions, and
external models already in production. Grid is added incrementally.

**Key constraints:**
- API keys MUST remain the same (zero user credential change)
- Rate limiting and metering must keep working throughout
- External models must remain rate-limited at all times

#### Phase 0 — Today (single cluster, no Grid)

```
Consumer → MaaS Gateway → external API (ExternalModel CR)
                        → local EPP → vLLM
```

Each site runs a standalone MaaS deployment:
- Praxis AI gateway with direct upstream clusters
- API key auth via maas-api
- Rate limiting via Kuadrant/Limitador (per-user, per-model)
- Metering via metering-service
- External models via `ExternalModel` CR (rate-limited by Limitador)
- No Grid awareness

#### Phase 1 — Add Grid alongside existing MaaS

```
Consumer → Grid Gateway → picks site → MaaS Gateway → external API
                                                     → local EPP → vLLM
```

- Install Grid Operator + Grid Gateway on each cluster
- Grid Gateway uses `api_key_auth` calling maas-api (same auth as today)
- Grid handles inter-site routing only (geo fencing, site selection)
- **MaaS Gateway stays in the path for ALL traffic** — handles rate
  limiting, metering, credential injection, external models
- External providers stay at MaaS level (`ExternalModel` CR) — rate
  limiting and metering continue to work unchanged
- Grid `token_rate_limit` is **disabled** — MaaS/Limitador handles limits
- Grid overlay starts with `InferenceProvider` CRs for local models only
- Shared DB (RDS) across sites for key/subscription consistency

#### Phase 2 — Enhanced Grid integration

- maas-api validation response extended with region + budget metadata
- Grid Gateway does geo fencing based on maas-api response (not JWT claims)
- MaaS → Grid auto-reconciliation controller: `MaaSModelRef` creation
  auto-generates `InferenceProvider` CRs in the Grid
- Grid Operators gossip and poll signals across sites
- MaaS rate limiting continues — still the single enforcement point

#### Phase 3 — Full multi-cluster

- All sites enrolled in the Grid mesh
- Consumers enter through any site's Grid Gateway
- Grid makes cost-aware, load-aware, geo-aware routing across all capacity
- Signals polling provides real-time load visibility across sites
- MaaS handles per-model limits, metering, external providers at each site

#### Phase 4 — Grid-native external models + rate limiting (optional)

Only when cross-site global budgets become a requirement:

- Enable Grid `token_rate_limit` with Valkey (per-subject, cross-site)
- Create `InferenceProvider` CRs with `backendKind: api_provider` for
  external APIs — Grid routes directly with its own limits and metering
- MaaS rate limiting continues for per-model limits (complementary)
- Grid: "alice can use 500K tokens/day total across all sites"
- MaaS: "team-a can use 1M tokens/month on claude-sonnet on this site"
- Retire MaaS-level `ExternalModel` config once Grid-level is validated

### Customer Impact During Migration (Scenario B)

**Key requirement: API keys MUST remain the same.** A tenant user should
not need to regenerate or reconfigure their API key when the platform
moves from single-cluster to multi-cluster. The key was minted by
maas-api, stored as a hash in the DB — as long as the DB is shared
(Phase 1) or replicated (Phase 2+), the same key validates on any site.

**What changes for the user:**

| Concern | Single cluster (Phase 0) | Multi-cluster (Phase 1+) | Impact |
|---------|-------------------------|-------------------------|--------|
| API key | `sk-oai-abc123...` | Same key | **None** |
| Base URL | `https://ai-gateway.site-a.example.com` | `https://ai-gateway.example.com` (DNS) | **Config change** if no stable DNS |
| Models available | Site-local only | All sites' models | Transparent improvement |
| Rate limits | Per-model (MaaS) | Same | **None** |
| Metering | Site-local | Same (shared DB) | **None** |

**URL migration strategy:**

- **With DNS (recommended)**: Put a stable CNAME (`ai-gateway.example.com`)
  in front from day one, even in single-cluster. When Grid goes live, DNS
  routes to the nearest site's Grid Gateway. Users never change their URL.

- **Without DNS**: Users need to update `ANTHROPIC_BASE_URL` /
  `OPENAI_BASE_URL` once. Minimize by setting up stable DNS before
  migration. Old URL can keep working during a transition period (old
  MaaS gateway stays active, Grid Gateway sits in front).

**Migration checklist for zero-disruption:**
1. Set up stable DNS name before Grid rollout
2. Verify API keys work on the new endpoint (shared DB)
3. Communicate URL change (if DNS wasn't pre-staged) with transition
   period where both old and new URLs work
4. Old MaaS gateway remains active — becomes the site-local gateway
   behind the Grid Gateway

### Admin Tooling and Multi-cluster Distribution

The tenant admin does not interact with clusters directly. The
management stack is:

```
Tenant Admin → Admin UI → Management API → GitOps / ACM / Ansible → clusters
```

- **Tenant Admin UI**: Web interface for model deployment, quota
  configuration, and usage dashboards. **Provided by RHCE** (Red Hat
  Cloud Extensions).
- **Management API**: Orchestration layer between the UI and the
  multi-cluster distribution system. Also serves quota and usage
  queries against the shared DB. **Provided by RHCE**.
- **Tenant User UI**: Self-service portal for model discovery, API
  key management, and personal usage. Can be a standalone app or a
  Red Hat Developer Hub / Backstage plugin.
- **GitOps / RHCE / ACM / Ansible**: Red Hat Cloud Extensions and
  OpenShift Advanced Cluster Management distribute MaaS CRDs
  (MaaSModelRef, ExternalModel, MaaSSubscription, MaaSAuthPolicy)
  to target clusters. The admin declares intent in the UI, the
  pipeline pushes CRDs to the right clusters.
- **MaaS → Grid reconciler** (to build): On each cluster, watches
  for ready MaaSModelRefs and auto-creates `InferenceProvider` CRs
  so models become grid-routable without manual Grid CRD management.

### Decision: All clusters are equivalent — no special hub deployment

Every cluster runs the identical stack:

```
Every cluster:
  Grid Gateway + Grid Operator          (data + control plane)
  MaaS Gateway + maas-controller        (site-local enforcement)
  maas-api + metering-service           (shared DB)
  Tenant Admin UI + Management API      (stateless)
  Tenant User UI                        (stateless)
```

The "hub" is a **DNS designation**, not a deployment difference:
- `admin.example.com` → Admin UI (any cluster)
- `portal.example.com` → User UI (any cluster)
- `ai-gateway.example.com` → nearest Grid Gateway (geo-routed)

**Rationale**:
- **No special snowflake** — ops deploys the same stack everywhere
- **HA/DR** — hub goes down, repoint DNS, everything works
- **UIs are stateless** — they call APIs, APIs read shared DB/overlay.
  Any cluster can serve any UI
- **Consistent with Grid design** — Grid already treats all sites as
  equivalent for routing and enforcement
- The Grid hub (SWIM seed) and the management hub (user-facing DNS)
  are the same cluster by convention, but can be split if needed

### Observability — ACM Multi-cluster Observability

Metrics aggregation across clusters uses the **OpenShift Multi-cluster
Observability Operator** (part of ACM):

```
Cluster A (Prometheus) ──┐
                         ├── Thanos (ACM) ── Grafana dashboards
Cluster B (Prometheus) ──┘
```

- Each cluster's components (MaaS gateway, Grid Operator, metering-service,
  Limitador) emit **Prometheus metrics**
- ACM's **Multi-cluster Observability Operator** collects and aggregates
  into a central **Thanos** instance
- **Grafana dashboards** provide cross-cluster views: token usage per user,
  model latency, cost breakdown, rate limit utilization, Grid routing
  decisions
- The **Tenant Admin UI** links to Grafana for usage dashboards rather
  than building custom visualization — reuse existing observability infra

This replaces the need for a custom cross-cluster metering aggregation
system for observability purposes. Metering-service still handles the
transactional data (API key → usage → quota enforcement) via the shared
DB, but the dashboards and alerting use the Thanos/Grafana stack.

### Open questions

- **Enrollment automation**: Manual token minting OK for dogfood?
  Or automate for scale?
- **MaaS → Grid auto-reconciliation**: When a tenant admin creates a
  `MaaSModelRef` + `MaaSSubscription`, should a controller auto-create
  the corresponding `InferenceProvider` CR? Critical for smooth upgrade
  path — deploy a model in MaaS, it automatically becomes grid-routable.

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
| Configure per-user quota | `PATCH /api/v1/admin/quotas/{username}` on metering-service | **To build** |
| Get all users' usage | `GET /api/v1/admin/usage` on metering-service | **To build** |

**Auth**: K8s RBAC for CRDs, offline token for metering-service APIs.

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
| Get personal usage | Self-service usage endpoint on metering-service | **Needs design** |
| View remaining quota | Self-service quota endpoint on metering-service | **Needs design** |

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

- **Self-service usage/quota APIs**: maas-api handles model listing and
  key management for tenant users. Metering-service handles usage/quota
  for admins. Need a user-scoped subset — auth via API key → username
  resolution (maas-api already does this in validation).

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

**Context**: Two rate limiting implementations exist:
- MaaS: Kuadrant/Limitador, per-user per-subscription per-model, CRD-driven,
  fixed window, counts actual LLM tokens from response body
- Grid: Praxis `token_rate_limit`, per-subject across all models,
  sliding window or token bucket, in-process with reserve/reconcile

Both count LLM tokens (not HTTP requests). They are complementary in
scope but running both risks double-counting and adds complexity.

**Decision**: For the MVP, disable `token_rate_limit` at the Grid Gateway.
Rate limiting stays at the MaaS level only (Kuadrant/Limitador).

**Rationale**:
- MaaS rate limiting is already working and CRD-driven — zero new work
- Avoids double-counting (only one layer deducts from budgets)
- Grid's `token_rate_limit` needs Valkey for multi-replica accuracy,
  and CRD-driven config generation doesn't exist yet
- The Praxis team has an open design question about the Kuadrant
  relationship (ai#127) — let that settle before taking a dependency

**What MaaS rate limiting gives us for the MVP**:
- Per-user, per-model token limits (from `MaaSSubscription.tokenRateLimits`)
- CRD-driven — admin creates subscription, limits auto-apply
- Shared Limitador state within a site (single instance)

**What we defer to post-MVP**:
- Cross-site global budget enforcement (alice uses tokens on site-A
  and site-B, combined limit). Requires either shared Limitador state
  across sites or Grid's `token_rate_limit` with Valkey
- Reserve/reconcile (Grid's smarter approach that blocks over-budget
  requests before they hit the backend)
- Per-subject global budget across all models (Grid's scope)

**Migration path**: When cross-site budgets become a requirement, enable
Grid's `token_rate_limit` with Valkey backend for global per-subject
enforcement. MaaS rate limiting continues for fine-grained per-model
limits. The two layers become complementary:
- Grid: "alice can use 500K tokens/day total across all sites"
- MaaS: "team-a can use 1M tokens/month on claude-sonnet on this site"

---

## Decision: Auth migration — API key at the Grid level via maas-api

**Context**: The v3 Grid demo uses JWT-only auth (HS256, local signature
verification). Existing MaaS deployments use API key auth via maas-api
(`api_key_auth` filter → `POST /internal/v1/api-keys/validate`). Users
have API keys today and must keep using them.

**Decision**: Use the `api_key_auth` filter at the Grid Gateway level,
integrated with maas-api. Extend maas-api's validation response to
return Grid-relevant metadata (region, budget).

### How it works

The Grid Gateway runs the same `praxis-ai-proxy` binary and has access
to all filters. The combined filter chain:

```
api_key_auth → model_to_header → intelligent_route → token_rate_limit → token_count → load_balancer
```

1. `api_key_auth` calls maas-api `/internal/v1/api-keys/validate`
2. maas-api returns: username, groups, **region** (from tenant config),
   **token budget** (from subscription rate limits)
3. `api_key_auth` (or a follow-up filter) promotes region and budget to
   headers (e.g., `X-Grid-Region`, `X-Grid-Rate`)
4. `intelligent_route` reads region from header for geo fencing (instead
   of JWT `grid_region` claim)
5. `token_rate_limit` uses `authenticated_subject` for per-user budget
6. Rest of the chain works unchanged

### What needs to change

**maas-api**: Extend the validation response to include:
- `region` — sourced from `AITenant` or `MaasTenantConfig` (the tenant's
  allocated geo region)
- `token_budget` / `rate_limits` — sourced from the matched
  `MaaSSubscription.tokenRateLimits`

**Praxis AI**: The `api_key_auth` filter (or a new lightweight filter)
needs to map the extended validation response fields into headers that
`intelligent_route` and `token_rate_limit` can consume. Alternatively,
`intelligent_route` could accept a header-based region source alongside
the existing JWT claim-based `match_claims`.

### Why this approach

- **Users keep their API keys** — zero credential change
- **No JWT infrastructure needed** — no Keycloak, no token exchange,
  no new credential format for users to learn
- **Same auth flow as today's dogfood** — `api_key_auth` + maas-api is
  proven in production. Just extended with metadata
- **Grid features work** — geo routing and token budget operate on
  identity + metadata from maas-api, not from JWT claims
- **JWT can be added later** as an additional auth path for
  service-to-service or automation use cases (dual-stack)

### Rejected alternatives

- **JWT only (v3 approach)**: Requires all users to get JWTs instead
  of API keys. Breaks existing users. Needs IdP infrastructure.
- **API key → JWT exchange**: User presents API key, gets back a JWT.
  Adds complexity (exchange endpoint, token refresh). Unnecessary if
  maas-api can return the needed metadata directly.
- **Dual-stack from day one**: Supporting both API keys and JWTs adds
  testing surface. Start with API keys (existing users), add JWT
  later when there's a real need (automation, cross-org federation).

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
