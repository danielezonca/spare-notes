# Praxis AI Tokenomics Strategy

Analysis of the Praxis team's current and planned capabilities around
token accounting, rate limiting, metering, and cost attribution.
Source: [Epic #656](https://github.com/praxis-proxy/ai/issues/656)
and linked issues.

---

## The Three Epics

The tokenomics space is covered by three overlapping epics, all targeting
v1.0.0 or unversioned:

### Epic #656 — Tokenomics (umbrella)
The full lifecycle: counting → metering → rate limiting → cost attribution.
Success criteria:
- Token counts from all providers (streaming + non-streaming)
- Token count estimations (pre-forward)
- Token-denominated rate limiting with reserve/reconcile
- Accurate metering for billing + chargeback
- **Cross-cluster token quota enforcement**

### Epic #121 — Token Rate Limiting
Deep dive on the rate limiting subsystem:
- Token bucket + sliding window algorithms
- Composite bucket keys (per-user, per-model, per-header, composites)
- Hierarchical quotas (org > team > user)
- Distributed counters (in-memory or Valkey)
- Response headers (Retry-After, X-RateLimit-*)

### Epic #78 — AI Cost and Budget Controls
Cost attribution and budget enforcement:
- Input token estimation before forwarding
- Per-client/per-route token budgets
- Cost attribution: user, session, model, endpoint
- Budget exceeded behavior: reject (429) or queue

---

## What's Shipped (already merged)

| Feature | Issue | What it does |
|---------|-------|-------------|
| **Sliding window + token bucket** | #796 | Core algorithms for `token_rate_limit` filter. M1 (sliding window), M2 (token bucket), M6 (response headers) |
| **Shared Valkey backend** | #790 | Multi-replica state via Valkey. Lua scripts for atomic reserve/reconcile |
| **Configurable estimation strategies** | #1008 | Four strategies for pre-forward reservation: `fixed`, `max_tokens`, `input_plus_max_tokens`, `model_scaled` |
| **Token-type weights** | #1132 | Weight input/output/reasoning tokens differently at reconciliation (e.g., output tokens cost 3x input) |

**Current state**: The `token_rate_limit` filter is functional but
feature-gated (`--features token-rate-limit-filter`). It supports
sliding window and token bucket, in-memory or Valkey state, per-subject
keying, configurable estimation, and token-type weighting.

---

## What's In Flight / Planned

### v0.4.0 milestone

| Feature | Issue | Status |
|---------|-------|--------|
| **Replace Valkey Lua with plain commands** | [#1378](https://github.com/praxis-proxy/ai/issues/1378) | Open — Lua scripts must go before the filter leaves experimental. Moving to INCRBY counters + WATCH/MULTI/EXEC |

### v0.5.0 milestone

| Feature | Issue | Status |
|---------|-------|--------|
| **Compositional bucket keys (M5)** | [#1334](https://github.com/praxis-proxy/ai/issues/1334) | Open — `global`, `authenticated_subject`, `ip`, `model`, `header`, or ordered composites. Missing dimension fails closed |
| **Pre-forward token ceiling** | [#1332](https://github.com/praxis-proxy/ai/issues/1332) | Open — Reject requests exceeding input/output token limits before forwarding. Uses tiktoken tokenizer |
| **API key validation filter** | [#707](https://github.com/praxis-proxy/ai/issues/707) | Open — HTTP callout to external key service (maas-api). In-memory cache. Writes to filter_metadata. Key stripping + credential injection |

### v0.6.0 milestone

| Feature | Issue | Status |
|---------|-------|--------|
| **external_metering filter** | [#577](https://github.com/praxis-proxy/ai/issues/577) | Open — Pre-request balance check + post-response CloudEvent usage reporting. Fire-and-forget async. Fail-open configurable |
| **First-party cost dashboards** | [#1123](https://github.com/praxis-proxy/ai/issues/1123) | Open — Per-tenant metric labels + Grafana wiring. Access-controlled so tenants see only own consumption |

### Unversioned (no milestone)

| Feature | Issue | Status |
|---------|-------|--------|
| **Simultaneous per-user + per-model limits** | [#979](https://github.com/praxis-proxy/ai/issues/979) | Open — A single request must consume BOTH user budget and model budget. Most restrictive wins. Not achievable safely with current single-rule-match |
| **Soft/shadow enforcement** | [#1381](https://github.com/praxis-proxy/ai/issues/1381) | Open — Per-rule `enforcement: hard|soft|shadow`. Soft forwards with annotation. Shadow observes would-deny. Metrics distinguish modes |
| **Reasoning tokens in CloudEvents** | [#1201](https://github.com/praxis-proxy/ai/issues/1201) | Open — Emit reasoning_tokens + configurable provider field in usage events |
| **Entitlement claim fencing** | [#1283](https://github.com/praxis-proxy/ai/issues/1283) | Open — `match_claims` on intelligent_route candidates. Geo/tier/sovereignty fencing via JWT claims. Fails closed |
| **WebSocket upgrade bypass** | [#1359](https://github.com/praxis-proxy/ai/issues/1359) | Open — Upgrade tunnels bypass all body-parsing filters (metering, token_count, guardrails). Real production issue discovered in metering deployment |

---

## Architecture: The Token Lifecycle

Based on the issues, the Praxis team envisions this pipeline:

```
Request arrives
  │
  ├── 1. AUTH: api_key_auth (#707) or policy (JWT)
  │      → resolves identity (subject, groups, subscription)
  │
  ├── 2. CEILING: token_ceiling (#1332)
  │      → rejects if input/output tokens exceed max before forwarding
  │
  ├── 3. RATE LIMIT: token_rate_limit (reserve phase)
  │      → estimates token cost (4 strategies)
  │      → reserves from per-subject/model/composite budget
  │      → rejects with 429 if budget exhausted
  │      → supports hard/soft/shadow enforcement (#1381)
  │
  ├── 4. METERING: external_metering (#577, pre-request)
  │      → balance check against external service
  │      → optional second enforcement layer
  │
  ├── 5. ROUTE: intelligent_route + load_balancer
  │      → match_claims fencing (#1283)
  │      → send to provider
  │
  ├── 6. COUNT: token_count (response phase)
  │      → extracts actual usage from provider response
  │      → handles streaming SSE (#770, #1360)
  │
  ├── 7. RATE LIMIT: token_rate_limit (reconcile phase)
  │      → actual vs estimated: refund or charge overage
  │      → token-type weights applied (#1132)
  │
  └── 8. METERING: external_metering (#577, post-response)
         → CloudEvent with usage data (fire-and-forget)
         → includes reasoning tokens (#1201)
         → billing/chargeback data pipeline
```

---

## Key Design Decisions in the Praxis Team

### 1. Reserve/reconcile is the core admission model

Not just "check balance then proceed" — tokens are **reserved upfront**
at admission time, preventing concurrent requests from overshooting.
After the response, actual usage is reconciled against the estimate.
This is fundamentally different from the Pricetag dogfood's coarse
yes/no balance check.

### 2. Compositional bucket keys (#1334) solve the multi-scope problem

Instead of hardcoding "per-user" or "per-model", bucket keys are
composable: `[authenticated_subject, model]` creates per-user-per-model
buckets. `[model]` alone creates per-model global buckets. This is
the building block for #979 (simultaneous user + model limits).

### 3. external_metering is about data, not enforcement

The design (#577) explicitly says the filter is for **usage reporting
and balance checks** — it's the data pipeline to external billing
systems, not the primary enforcement mechanism. Rate limiting
enforcement is `token_rate_limit`. The balance check in
`external_metering` is a secondary safety net, fail-open by default.

### 4. Soft/shadow enforcement (#1381) enables safe rollout

Rules can be `soft` (allow but annotate) or `shadow` (observe only).
This lets operators deploy token budgets in shadow mode first, verify
the would-deny rate matches expectations, then flip to hard enforcement.
Critical for production rollouts.

### 5. API key filter (#707) has an in-memory cache

The `api_key_auth` filter is designed with an **in-memory TTL cache**
to avoid per-request HTTP roundtrips to maas-api. This addresses the
latency concern with `external_metering`'s per-request callout. The
cache makes key validation almost free after the first lookup.

### 6. WebSocket bypass (#1359) is a real production gap

Discovered in a real metering deployment — WebSocket upgrades tunnel
past all body-parsing filters. The metering team found clients
(OpenCode 2.x) that default to WebSocket transport, creating
unmetered traffic. This affects not just metering but also guardrails
and token counting.

---

## Implications for Our Migration

### What aligns with our needs

| Need | Praxis capability | Status |
|------|------------------|--------|
| Per-user token budgets | `token_rate_limit` + `key: authenticated_subject` | Shipped |
| Multi-tier quotas (hourly + daily + monthly) | Multiple rules per filter | Shipped |
| Per-model limits | Compositional keys with `model` dimension (#1334) | v0.5.0 |
| Simultaneous user + model budgets | #979 (depends on #1334) | Planned |
| Pre-forward token ceiling | #1332 | v0.5.0 |
| API key auth with cache | #707 | v0.5.0 |
| Usage recording for billing | `external_metering` CloudEvents (#577) | v0.6.0 |
| Cross-cluster enforcement | Valkey backend | Shipped (Lua → plain commands in v0.4.0) |
| Soft rollout | Shadow/soft enforcement (#1381) | Planned |
| Geo fencing | match_claims (#1283) | PR open |

### What's not yet available

| Gap | Issue | When |
|-----|-------|------|
| Simultaneous per-user AND per-model in one request | #979 | After v0.5.0 (depends on M5 keys) |
| external_metering filter (usage recording) | #577 | v0.6.0 |
| API key validation filter | #707 | v0.5.0 |
| Cost dashboards (per-tenant Grafana) | #1123 | v0.6.0 |
| WebSocket bypass fix | #1359 | Unversioned |
| Valkey Lua replacement (blocks leaving experimental) | #1378 | v0.4.0 |

### Migration timing

The critical features for our Phase 1 migration land in **v0.5.0**:
- API key validation filter (#707) — needed for Grid Gateway auth
- Compositional bucket keys (#1334) — needed for per-model limits
- Pre-forward ceiling (#1332) — prevents runaway requests

The usage recording pipeline (`external_metering` #577) lands in
**v0.6.0**. Until then, the existing metering-service CloudEvent
pipeline continues to work (it's the same filter, just being
upstreamed into the ai repo).

### The Kuadrant question (ai#127)

The `token_rate_limit` module docs flag an open question about its
relationship to Kuadrant's `TokenRateLimitPolicy`. Based on the
roadmap, the Praxis team is building `token_rate_limit` as a
**self-contained replacement** — not an integration with Kuadrant.
The compositional keys (#1334), simultaneous multi-scope limits
(#979), and soft/shadow enforcement (#1381) all duplicate Kuadrant
capabilities in-process. This suggests the long-term direction is
Praxis-native rate limiting, with Kuadrant as an optional
compatibility layer during migration.
