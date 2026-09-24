# AI Grid — Deployment Scenarios

Two distinct paths to multi-cluster, depending on starting point.

---

## Scenario A — Greenfield / Target Architecture

No existing MaaS deployment. Grid is the platform from day one.

### Traffic flow

```
Consumer → Grid Gateway → picks site → site gateway → EPP → vLLM
                       → external API (api_provider)
```

### How each concern is handled

| Concern | How it works |
|---------|-------------|
| **Auth** | API key via maas-api (`api_key_auth` filter). Optionally also JWT from IdP for service-to-service. API key management (create, revoke, list) via maas-api remains the user-facing credential system |
| **Rate limiting** | Grid `token_rate_limit` with Valkey (per-subject, cross-site global) |
| **External models** | Grid `InferenceProvider` with `backendKind: api_provider`. Grid routes directly, enforces limits and meters |
| **Observability** | Prometheus metrics + Thanos/Grafana dashboards |
| **Model catalog** | `InferenceProvider` CRs declare models. Overlay renders candidates |
| **Key management** | maas-api provides key CRUD, validation, subscription binding. This is not a MaaS-vs-Grid concern — API key management is a platform service used by both scenarios |

Even in greenfield, maas-api is the API key management layer. Building
a separate key management system for Grid would be redundant — maas-api
already handles creation, hashing, validation, subscription binding,
revocation, and expiry. The Grid Gateway calls `api_key_auth` →
maas-api on every request, same as the brownfield path.

### Characteristics

- Simplest architecture — one routing layer, one auth model, one rate
  limiting engine
- Requires building out Grid's management APIs (usage dashboards) —
  model listing and key management already provided by maas-api in both
  scenarios
- No migration concerns — no existing users to break
- Grid handles everything: routing, limits, external providers, metering

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

### Phase 1 — Add Grid alongside existing MaaS

```
Consumer → Grid Gateway → picks site → MaaS Gateway → external API
                                                     → local EPP → vLLM
```

What happens:
- Install Grid Operator + Grid Gateway on each cluster
- Grid Gateway uses `api_key_auth` calling maas-api (same auth as today)
- Grid handles inter-site routing only (geo fencing, site selection)
- **MaaS Gateway stays in the path for ALL traffic** — handles rate
  limiting, credential injection, external models
- External providers stay at MaaS level (`ExternalModel` CR) — rate
  limiting continues to work unchanged
- Grid `token_rate_limit` is **disabled** — MaaS/Limitador handles limits
- Grid overlay starts with `InferenceProvider` CRs for local models only
- Shared DB across sites for key/subscription consistency

What doesn't change for users:
- Same API key
- Same models work
- Same rate limits apply
- URL changes only if stable DNS wasn't pre-staged

### Phase 2 — Enhanced Grid integration

- maas-api validation response extended with region + budget metadata
- Grid Gateway does geo fencing based on maas-api response (not JWT claims)
- MaaS → Grid auto-reconciliation controller: `MaaSModelRef` creation
  auto-generates `InferenceProvider` CRs in the Grid
- Grid Operators gossip and poll signals across sites
- MaaS rate limiting continues — still the single enforcement point
- maas-api reads Grid overlay for multi-cluster model listing

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

### All clusters are equivalent

Every cluster runs the identical stack:

```
Every cluster:
  Grid Gateway + Grid Operator          (data + control plane)
  MaaS Gateway + maas-controller        (site-local enforcement)
  maas-api                              (shared DB)
  Tenant Admin UI + Management API      (stateless, provided by RHCE)
  Tenant User UI                        (stateless)
```

The "hub" is a **DNS designation**, not a deployment difference:
- `admin.example.com` → Admin UI (any cluster)
- `portal.example.com` → User UI (any cluster)
- `ai-gateway.example.com` → nearest Grid Gateway

**Rationale**:
- No special snowflake — ops deploys the same stack everywhere
- HA/DR — hub goes down, repoint DNS, everything works
- UIs are stateless — any cluster can serve them
- Consistent with Grid design — all sites are equivalent

### Shared DB: now, replicated later

**Phase 1 (shared DB):** All sites connect to the same PostgreSQL
instance. Keys and subscriptions are immediately consistent.
Simple, no replication logic. Works when sites are same-region.

**Phase 2+ (hub + reconciler):** When sites span regions or cloud
boundaries, hub owns the write path. A reconciler syncs key hashes
and subscription data to peer sites on a multi-second interval.
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
| **Auth** | api_key_auth → maas-api | api_key_auth → maas-api (same) |
| **Rate limiting** | Grid `token_rate_limit` + Valkey | MaaS Limitador only |
| **External models** | Grid `InferenceProvider` api_provider | MaaS `ExternalModel` CR |
| **Metering** | Grid-level | MaaS-level (Thanos for dashboards) |
| **Model catalog** | Grid overlay only | Grid overlay + MaaS informer |
| **Existing users** | None | Must preserve keys, URLs, limits |
| **Complexity** | Simpler architecture | More moving parts, phased migration |
| **When to choose** | New platform, no legacy | Existing MaaS users in production |
