# Praxis Token Rate Limiting Demos — Analysis

Analysis of experimental demos from
[praxis-proxy/experimental/demos](https://github.com/praxis-proxy/experimental).
These show the evolving config shape and capabilities across three
stages of maturity.

---

## Demo Overview

| Demo | What it proves | Auth | State | Grid | Status |
|------|---------------|------|-------|------|--------|
| `token-rate-limit-per-app-budgets` | Per-app budgets, mixed algorithms (sliding window + token bucket) | None (header only) | Valkey (shared) | No | Fork branch |
| `grid-distributed-token-rate-limit` | Distributed quota + Grid routing across 3 clusters | Basic Auth | Valkey (shared) | Yes | Dev branches |
| `grid-cloud-burst` | Regional failover + cloud overflow + soft quotas | Basic Auth | In-memory | Yes | Dev branches |
| `ai-gateway` | Production-like personal gateway with per-model budgets | JWT (policy) | In-memory | No | Released config |

---

## 1. `token-rate-limit-per-app-budgets`

Per-app budgets with two rules using different algorithms in one filter.

**Filter chain**: `router → token_rate_limit → token_count → access_log → load_balancer`

**Rules**:

| Rule | Match | Algorithm | Window | Capacity | Reserve | Key |
|------|-------|-----------|--------|----------|---------|-----|
| gold-tier | `x-tier: gold` | sliding_window | 10s | 40 tokens | 15 | `x-app-id` header |
| silver-tier | `x-tier: silver` | token_bucket | — | 10, refill 2/s | 7 | `x-app-id` header |

**Key insights**:
- Per-rule algorithm choice (first match wins)
- `bucket_key_header` creates independent budgets per app
- Shared Valkey across 2 gateway replicas — budget is cross-replica
- Reserve/reconcile confirmed: estimate held at admission, actual reconciled post-response
- Recovery difference: sliding_window recovers when entries slide out (flat), token_bucket refills continuously (smooth)

---

## 2. `grid-distributed-token-rate-limit`

Authenticated token quotas shared across gateways + Grid multi-cluster routing.

**Architecture**:
```
Client → Consumer A or B → Valkey (shared quota) → Grid overlay
                                                    → Provider west/central/east
                                                    → VCR backend
```

**Rate limit**:
- **Key**: composite `authenticated_subject` + `model`
- **Algorithm**: sliding_window, 60-token window
- **Backend**: Valkey shared across both consumer gateways
- **Admission before routing**: quota check BEFORE provider selection — 429 never reaches a provider

**Key insights**:
- Quotas follow users, not replicas
- Grid routing independent of quota state (overlay is async)
- Fail-closed on Valkey outage (503, not silent admission)
- Full multi-cluster: 3 Kind clusters, Grid operator + overlay-sync on each, mTLS
- Basic Auth used as JWT stand-in (same metadata contract)

---

## 3. `grid-cloud-burst`

The most advanced demo — regional failover, cloud overflow, soft quotas.

**Topology**:
```
Consumer East/West → [East providers (llm-d-east-1/2)]
                   → [West providers (llm-d-west-1/2)]
                   → [Azure OpenAI + OpenAI] (cloud overflow)
```

**Filter chain**: `basic_auth → token_rate_limit → json_body_field → headers → intelligent_route → token_count → load_balancer`

**Rate limit config** (most sophisticated):

```yaml
- filter: token_rate_limit
  enforcement: soft          # over-budget → 200 + annotation, NOT 429
  key:
    principal:
      source: metadata
      name: identity.user_id
      onMissing: reject
    model:
      source: header
      name: x-model
      onMissing: reject
      allowedModels: ["Qwen/Qwen3-0.6B"]
  reservationTimeout: 2m
  limits:
    maxKeys: 32
    maxActiveReservations: 16
  rules:
    - name: alice
      match: { metadata: { identity.user_id: alice } }
      estimation: { strategy: fixed, tokens: 5 }
      token_budgets:
        - window: 1m
          capacity: 60
    - name: bob
      match: { metadata: { identity.user_id: bob } }
      estimation: { strategy: fixed, tokens: 5 }
      token_budgets:
        - window: 10m
          capacity: 5000
```

**New features not in other demos**:
- **`enforcement: soft`**: over-budget served (200) with `x-ratelimit-governance: over_allocation` header. Flip to `hard` for 429
- **Composite key with `principal` + `model`**: both from auth metadata + header
- **`allowedModels`**: restricts valid model names for rate limiting
- **`token_budgets[]` (plural)**: supports multiple windows per rule (multi-tier)
- **`estimation.strategy: fixed`**: explicit estimation config
- **`limits`**: cardinality controls (maxKeys, maxActiveReservations)
- **`reservationTimeout: 2m`**: explicit timeout for abandoned reservations
- **Match on metadata, not headers**: `identity.user_id` from auth filter, not client-supplied

---

## 4. `ai-gateway`

Production-like personal gateway. Three listeners with per-model budgets.

**Rules (OpenAI listener :8080)**:

| Rule | Match | Window | Capacity | Reserve | Timeout |
|------|-------|--------|----------|---------|---------|
| agent-daily (Qwen 27B) | `x-model: qwen3.8:27b` | 24h | 10M | 10,000 | 300s |
| agent-daily (Qwen Coder) | `x-model: qwen3-coder:30b` | 24h | 10M | 10,000 | 300s |
| agent-daily (DeepSeek R1) | `x-model: deepseek-r1:32b` | 24h | 10M | 10,000 | 300s |
| gpt-4o | `x-model: gpt-4o` | 1m | 40,000 | 2,000 | 120s |
| gpt-4o-mini | `x-model: gpt-4o-mini` | 1m | 40,000 | 1,000 | 120s |
| free (catch-all) | — | 1m | 5,000 | 200 | — |

**Key insights**:
- Daily budgets for local models (protects GPU queue)
- Per-minute budgets for hosted models (protects against billing surprises)
- `reserved_tokens: 10000` is measured, not guessed — "one trivial Codex turn costs 9,471 tokens"
- `reservation_timeout: 300s` for large models (34-78s generation, default 30s too short)
- Catch-all rule required — unmatched requests are not limited at all
- Known gap (grid#101): JWT `sub` published by policy but no budget rule can key on it yet
- Backend: `memory` (single instance, not shared)

---

## Config Shape Evolution

The demos show an evolving config API:

| Generation | Demo | Key features |
|-----------|------|-------------|
| **Gen 1** (released) | ai-gateway | `reserved_tokens`, `algorithm`, `window`, `capacity`, match on headers |
| **Gen 2** (fork) | per-app-budgets | `estimate_tokens`, `bucket_key_header`, per-rule algorithm choice |
| **Gen 3** (dev branches) | grid-distributed | Composite `key` with `principal`+`model` sources, `match.metadata` |
| **Gen 4** (dev branches) | grid-cloud-burst | `enforcement: soft`, `estimation.strategy`, `token_budgets[]`, `limits`, `allowedModels` |

Gen 4 is the most complete and addresses all the advanced requirements:
multi-tier quotas, soft enforcement, composite keys, per-user rules,
cardinality controls, and metadata-based matching.

---

## Feature Matrix

| Feature | per-app | grid-distributed | grid-cloud-burst | ai-gateway |
|---------|:---:|:---:|:---:|:---:|
| Sliding window | ✓ | ✓ | ✓ | ✓ |
| Token bucket | ✓ | — | — | — |
| Valkey shared state | ✓ | ✓ | — | — |
| In-memory state | — | — | ✓ | ✓ |
| Reserve/reconcile | ✓ | ✓ | ✓ | ✓ |
| Composite key (principal+model) | — | ✓ | ✓ | — |
| Soft/shadow enforcement | — | — | ✓ | — |
| Multi-window per rule | — | — | ✓ | — |
| Per-user rules | — | — | ✓ | — |
| Grid routing | — | ✓ | ✓ | — |
| mTLS to providers | — | ✓ | ✓ | — |
| Cloud overflow | — | — | ✓ | — |
| Daily budgets | — | — | — | ✓ |
| Multiple listeners | — | — | — | ✓ |
| Authentication | — | Basic | Basic | JWT |

---

## Implications for Migration

### What these demos prove is production-ready (or near)

1. **Reserve/reconcile** works across all demos — the admission model is solid
2. **Valkey shared state** works across replicas (per-app, grid-distributed)
3. **Composite keys** (principal+model) work — needed for per-user-per-model limits
4. **Grid integration** — quota admission happens before routing, Grid overlay is independent
5. **Soft enforcement** — safe rollout with shadow/soft mode before hard 429s

### What's still in dev branches (not merged upstream)

- The Gen 3/4 config shapes (composite `key`, `enforcement`, `token_budgets[]`, `limits`)
- These are ahead of the released `praxis-ai` config API
- The roadmap issues (#1334 compositional keys, #1381 soft enforcement) track upstreaming these features

### Practical guidance from ai-gateway demo

- `reserved_tokens: 10000` is realistic for Claude Code / Codex (system prompts + tool schemas = ~9,471 tokens baseline)
- `reservation_timeout: 300s` needed for large model inference (default 30s too short)
- Catch-all rule is mandatory — forgetting it means some traffic is unlimited
- Daily + per-minute budgets serve different purposes (GPU protection vs billing protection)
