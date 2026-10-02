# AI Grid — Design Notes

Tracking architecture decisions, migration strategy, and open questions
for the multi-cluster Grid deployment.

---

## Architecture Overview

The AI Grid is a two-plane system deployed across **Region Ingress
Clusters** and **Inference clusters**:

- **Control plane**: Grid Operators at each site (ingress and inference),
  connected via SWIM gossip (membership, liveness) and signals
  polling (load metrics). Operators reconcile CRDs and generate a
  routing overlay ConfigMap. The `ai grid` reconciler watches
  opted-in llm-d deployments and creates `InferenceProvider` CRs.
  The `external model` reconciler (ingress clusters only) creates
  `ExternalModel` + `MaaSModelRef` CRs in MaaS from InferenceProvider
  state.
- **Data plane**: MaaS Gateways on ingress clusters are the customer
  entry point — they handle API key auth, rate limiting, metering,
  and credential injection. Grid Gateways on all sites handle
  routing + service-level auth (SA tokens, no user API key auth) — consuming the overlay to make model-based,
  geo-aware, cost-aware routing decisions and forwarding to local or
  remote standalone gateways serving llm-d backends.

### Cluster topology

Ingress clusters run MaaS Gateway + maas-api in addition to the shared
Grid stack. Inference clusters run inference workloads only and can
be disconnected from the public internet — only private network
connectivity to other clusters is required (for SWIM gossip, signals
polling, and data-plane mTLS). A single cluster can serve both roles
— there is no hard requirement to separate them. When both roles are
on one cluster, it runs the full stack. The distinction is about MaaS
presence, not model placement.

### Components — Ingress cluster

| Component | Role |
|-----------|------|
| MaaS Gateway (Praxis AI) | Customer entry point. API key auth, rate limiting, metering, credential injection |
| maas-api | API key validation, model listing, subscription management |
| maas-controller | CRD reconciliation (ExternalModel, MaaSModelRef, MaaSSubscription, etc.) |
| Grid Gateway (Praxis AI) | Routing only. model_to_header → intelligent_route → load_balancer. No auth |
| Grid Operator | SWIM gossip, metrics polling/scraping, overlay generation |
| `ai grid` reconciler | Watches opted-in llm-d deployments, creates InferenceProvider CRs (if hosting models) |
| `external model` reconciler | Creates ExternalModel + MaaSModelRef CRs from InferenceProvider state |
| Standalone Gateway | Serves llm-d models (optional — only if ingress cluster hosts models) |
| EPP + vLLM | Inference backend (optional — only if ingress cluster hosts models) |

### Components — Inference cluster

| Component | Role |
|-----------|------|
| Standalone Gateway | Serves llm-d models |
| EPP + vLLM | Inference backend |
| Grid Gateway (Praxis AI) | Receives cross-cluster traffic from ingress cluster Grid GW, routes to local standalone GW |
| Grid Operator | SWIM gossip, metrics polling/scraping, overlay generation |
| `ai grid` reconciler | Watches opted-in llm-d deployments, creates InferenceProvider CRs |

Inference clusters do NOT run MaaS Gateway, maas-api, maas-controller,
or the `external model` reconciler. They may be air-gapped from
the public internet but must maintain private network connectivity
to other clusters.

### Inter-site connections

- **Grid Gateways** route cross-site via mTLS
- **Grid Operators** gossip membership/liveness via SWIM (UDP, AES-GCM encrypted)
- **Grid Operators** poll each other's signals via `GET /v1/site/signals` (mTLS, Prometheus format)

---

## Decision: Unified ExternalModel abstraction

**Context**: MaaS Gateway is the customer entry point (auth, rate
limiting, metering). Grid Gateway handles routing only. All models
must be visible through MaaS regardless of where they run.

**Decision**: ALL models (llm-d internal + third-party) are registered
as `ExternalModel` CRs in MaaS. One `ExternalModel` per distinct model
name.

**How it works**:

1. **llm-d models**: The `ai grid` reconciler watches opted-in llm-d
   deployments and creates `InferenceProvider` CRs. The `external
   model` reconciler (ingress clusters only) creates `ExternalModel` CRs
   pointing to the local Grid GW endpoint
   (`grid-gw.grid-system.svc.cluster.local`). MaaS GW forwards to
   Grid GW, which routes to the right standalone GW (local or remote).

2. **Third-party APIs** (Anthropic, OpenAI, OpenRouter): Registered
   as `ExternalModel` CRs with the actual API endpoint (e.g.,
   `api.openai.com`). MaaS GW routes directly — Grid is bypassed
   because there is no cross-cluster routing decision to make.

**Rationale**:
- MaaS is first in the request path, so rate limiting and metering
  apply to ALL traffic regardless of backend type.
- llm-d models are on standalone gateways, not directly connected to
  MaaS GW. The `ExternalModel` CRD supports targeting any endpoint,
  including local services.
- One `ExternalModel` per distinct model name. Grid handles
  multi-site availability via multiple `InferenceProvider` CRs and
  the routing overlay.

**Traffic flow — llm-d models** (MaaS first, Grid routes):
```
Consumer → MaaS Gateway → Grid Gateway → picks target → standalone GW → EPP → vLLM
```

**Traffic flow — third-party APIs** (MaaS direct, Grid bypassed):
```
Consumer → MaaS Gateway → external API (rate-limited, metered)
```

**Post-MVP**: Grid-level external models (`InferenceProvider` with
`backendKind: api_provider`) remain an option for cost-aware
cross-site fallback routing.

---

## Decision: Model catalog is auto-populated

**Context**: How does the Grid know which models are available at
which site? And how does MaaS know about Grid-managed models?

**Decision**: The model catalog is auto-populated by two reconcilers.
Manual `InferenceProvider` creation remains possible but is not the
primary path.

**Reconciler chain**:
1. Platform admin deploys llm-d with opt-in annotation
   (`grid.praxis-proxy.io/managed: "true"`) in a filtered namespace.
2. `ai grid` reconciler (runs on each cluster with llm-d) watches
   annotated deployments and creates `InferenceProvider` CRs.
3. Grid Operator gossips provider state via SWIM. Overlay regenerates.
4. `external model` reconciler (runs on ingress clusters only) watches
   InferenceProviders and creates `ExternalModel` + `MaaSModelRef`
   CRs — one per distinct model name.

**Details**:
- Metrics scraping collects load signals (queue depth, KV cache), not
  model catalogs.
- Site matching uses `siteSelector.matchLabels` against `GridSite` CRs.
- Overlay renderer produces one routing candidate per (model, matched
  site) pair.
- Opt-in annotation + namespace filter prevent unauthorized models
  from entering the catalog.

### Cross-cluster model discovery via SWIM gossip

The Grid propagates model information across clusters. The
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

## Decision: Multi-cluster model listing via CRD materialization

**Context**: `GET /v1/models` on maas-api uses a K8s informer watching
local `MaaSModelRef` CRs. In multi-cluster, a user hitting Ingress A's
maas-api must see models from ALL clusters, not just Ingress A's local
models.

**Decision**: The `external model` reconciler (running on each ingress cluster)
creates `MaaSModelRef` CRs for every model in the Grid mesh. maas-api's
existing K8s informer sees these CRs — minor maas-api changes acceptable to simplify integration.

**How it works**:

```
GET /v1/models → existing K8s informer
  └── MaaSModelRef CRs on this ingress cluster
        ├── admin-created (manual ExternalModel for third-party APIs)
        └── reconciler-created (from InferenceProvider state via SWIM)
      → filter by user's subscriptions (existing logic)
      → return OpenAI-compatible model list
```

The `external model` reconciler watches `InferenceProvider` CRs, which
reflect the full mesh state thanks to SWIM gossip. For each distinct
model name, it creates an `ExternalModel` + `MaaSModelRef` CR on the
ingress cluster. Since the reconciler runs on every ingress cluster, each ingress cluster
independently converges to the same model catalog.

**Why this approach**:
- **Zero maas-api code changes** — uses the existing informer path
- No overlay mount, no composite lister, no new data path
- Subscription filtering works unchanged — user sees only models
  their subscription grants access to
- Each ingress cluster independently derives the catalog from SWIM state —
  no single point of failure, no cross-ingress-cluster CRD replication needed
  for model data

---

## Deployment Scenarios

See [deployment-scenarios.md](deployment-scenarios.md) for greenfield vs brownfield
migration with phased rollout, customer impact analysis, and
migration checklist.

### Open questions

- **Enrollment automation**: Manual token minting OK for dogfood?
  Or automate for scale?
- **What K8s resource does the `ai grid` reconciler watch?** Gateway
  CRD, InferencePool, or custom llm-d CRD? Needs confirmation with
  the llm-d team.

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

### Management plane topology

The `ai grid` reconciler watches llm-d deployments and creates
`InferenceProvider` CRs. The `external model` reconciler creates
`ExternalModel` + `MaaSModelRef` CRs on ingress clusters.

**Decision: Shared DB across ingress clusters + ACM/GitOps for CRD
replication.**

**API keys**: Shared PostgreSQL DB across all ingress clusters. Key
creation and validation are immediately consistent. Key revocation
is immediate. Inference clusters do not run maas-api and do not connect
to the shared DB.

**MaaS CRDs** (`MaaSSubscription`, `MaaSAuthPolicy`,
`MaasTenantConfig`, `AITenant`): Replicated across ingress clusters
via ACM/GitOps. Eventually consistent (seconds to low minutes).
These are admin-managed resources — propagation delay is invisible
for administrative operations.

**Auto-generated CRDs** (`ExternalModel`, `MaaSModelRef`): Created
independently on each ingress cluster by the `external model` reconciler.
Each ingress cluster sees the full mesh model catalog via SWIM gossip, so each
ingress cluster's reconciler converges to the same set of ExternalModel CRs.
Not replicated via ACM — generated locally from Grid state.

Rejected alternatives:
- **Centralized only (Option A)**: Cross-cluster latency on every validation
  call (hot path). Central node is SPOF for all sites.
- **Per-site isolated (Option B)**: Keys don't work across sites.
  Usage fragmented. Breaks the multi-cluster value proposition.
- **Grid-native replication (Option E)**: Leverages SWIM/CRDTs but
  is significant new work and mixes concerns between Grid (routing)
  and MaaS (management).

---

## Decision: Rate limiting — MaaS only for MVP, Grid later

See [auth-and-ratelimit.md](auth-and-ratelimit.md) for detailed comparison
of MaaS (Kuadrant/Limitador) vs Grid (Praxis token_rate_limit),
MVP decision rationale, and post-MVP migration path.

---

## Decision: Auth at MaaS level — Grid does routing

MaaS Gateway handles all authentication. Grid Gateway does routing
only — no `api_key_auth`, no identity resolution. The request flow
is: Customer → MaaS GW (auth, rate limiting, metering) → Grid GW
(routing) → target.

See [auth-and-ratelimit.md](auth-and-ratelimit.md) for the detailed
MaaS auth pipeline (Authorino, Limitador). Grid GW trusts requests
from MaaS GW based on network-level controls (same-cluster service
routing or cross-cluster mTLS).

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
