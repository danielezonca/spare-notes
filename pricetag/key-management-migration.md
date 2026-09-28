# API Key Management Migration: Pricetag → MaaS Subscriptions

Comparison of the Pricetag dogfood's key management model vs the full
MaaS subscription-based model, and the migration path between them.

---

## Data Model Comparison

### Pricetag (current dogfood)

```
API Key (PostgreSQL)
  ├── key_hash, prefix, name
  ├── username, group (flat labels)
  ├── subscription (passive label — for billing/reporting only)
  └── expires_at, ephemeral

Quota: single global env var (MONTHLY_TOKEN_QUOTA)
Model access: praxis.yaml config (model_access filter denylist)
Rate limiting: external_metering → metering-service balance check
```

### MaaS (target)

```
AITenant
  └── MaasTenantConfig

ExternalModel / LLMInferenceService
  └── MaaSModelRef (references backend)
        ├── MaaSSubscription (access grant + per-model token rate limits)
        │     └── generates → Kuadrant TokenRateLimitPolicy
        └── MaaSAuthPolicy (authorization: groups/users → models)
              └── generates → Kuadrant AuthPolicy

API Key (PostgreSQL)
  ├── key_hash, prefix, name
  ├── username, groups[] (snapshot at creation)
  ├── subscription (ACTIVE — bound at mint time, drives rate limits + auth)
  └── tenant, expires_at, ephemeral, labels, status
```

### Key Differences

| Concern | Pricetag | MaaS |
|---------|---------|------|
| **Subscription role** | Passive label (billing/reporting) | **Active** — drives rate limits, auth policies, model access |
| **Model access** | Config-based denylist (`model_access` filter) | CRD-based governance (`MaaSAuthPolicy` → Kuadrant AuthPolicy) |
| **Rate limiting** | Global env var + metering-service balance check | Per-subscription per-model CRD (`tokenRateLimits`) → Limitador |
| **Quota granularity** | Same for all users | Per-user, per-model, per-subscription, multiple windows |
| **Model readiness** | Always available (config-based) | Governance pairing required (subscription + auth policy) |
| **Group management** | Identity headers from caller | K8s RBAC + CRD subjects |
| **Key hashing** | bcrypt | SHA-256 (key_id embedded as per-key salt) |
| **Subscription binding** | Stored but not enforced | Bound at creation, immutable, fail-closed if missing |
| **Multi-window limits** | No (monthly only) | Yes (e.g., 50K/hour AND 1M/month per model) |
| **Cost tracking** | metering-service (LiteLLM pricing, CloudEvents) | Not in MaaS (separate concern) |
| **Dashboard** | Custom Go dashboard (metering-service) | Grafana (planned) |

---

## What Stays the Same

These work identically in both systems — no migration needed:

| Capability | How it works | Both systems? |
|-----------|-------------|---------------|
| **Key creation endpoint** | `POST /v1/api-keys` on maas-api | Same endpoint |
| **Key validation endpoint** | `POST /internal/v1/api-keys/validate` on maas-api | Same endpoint |
| **Key format** | `sk-oai-{key_id}_{secret}` | Same format |
| **Show-once pattern** | Plaintext returned once, only hash stored | Same |
| **Key revocation** | `DELETE /v1/api-keys/:id` | Same |
| **Key search** | `POST /v1/api-keys/search` | Same |
| **Ephemeral keys** | Short-lived keys with auto-cleanup | Same |
| **Subscription binding** | Key bound to subscription at creation | Same |

**maas-api is the same component in both systems.** The migration is
about what happens AROUND the key — how access, limits, and governance
are configured and enforced.

---

## What Changes

### 1. Subscription becomes active (passive label → enforcement driver)

**Pricetag**: Subscription is a string stored on the key, used for
billing reports. Rate limiting and model access are configured
separately (env var, praxis.yaml).

**MaaS**: Subscription is a CRD (`MaaSSubscription`) that actively
generates Kuadrant policies. Creating a subscription with
`tokenRateLimits` automatically creates a `TokenRateLimitPolicy`.
The subscription controls WHAT the user can access and HOW MUCH.

**Migration**: Tenant admin creates `MaaSSubscription` CRDs matching
existing subscription names. The rate limits in the CRD replace the
global `MONTHLY_TOKEN_QUOTA` env var. Keys already have the subscription
label — it just starts being enforced.

### 2. Model access moves from config to CRD

**Pricetag**: `model_access` filter in praxis.yaml with denylist +
override_groups. Changed by editing config and redeploying.

**MaaS**: `MaaSAuthPolicy` CRD maps subjects (groups/users) to models.
Changed by kubectl/GitOps. Generates Kuadrant AuthPolicy automatically.

**Migration**: Translate `model_access` denylist rules into
`MaaSAuthPolicy` CRDs. For each model × authorized group, create an
auth policy. The `model_access` filter can be removed once
MaaSAuthPolicy is enforced.

### 3. Rate limiting moves from metering-service to Kuadrant

**Pricetag**: `external_metering` filter → metering-service balance
check. Global quota, yes/no decision, no reservation.

**MaaS**: `MaaSSubscription.tokenRateLimits` → Kuadrant
`TokenRateLimitPolicy` → Limitador. Per-user per-model, CRD-driven,
multiple windows.

**Migration**: Create `MaaSSubscription` CRDs with `tokenRateLimits`
matching the desired budgets (replace the global env var with per-model
limits). Disable the `external_metering` balance check once Limitador
is enforcing.

### 4. Governance pairing is new

**Pricetag**: Models are always available. No "readiness" concept for
access control.

**MaaS**: A model only appears in `GET /v1/models` when it has BOTH a
subscription AND an auth policy targeting it. Without governance pairing,
the model stays `Pending`.

**Migration**: For every model that should be accessible, ensure there
is both a `MaaSSubscription` and a `MaaSAuthPolicy`. This is the most
operationally significant change — models without proper CRDs will
disappear from the API.

### 5. Auth mode for key creation

**Pricetag**: Key creation uses offline token + identity headers
(`X-MaaS-Username`, `X-MaaS-Group`). The caller (the onboarding service)
asserts the identity.

**MaaS**: Key creation requires OpenShift token (TokenReview). API
keys CANNOT create other API keys. Identity derived from the token,
not asserted by headers.

**Migration**: The onboarding flow needs to use OpenShift tokens
(or the SSO/OIDC bridge) instead of offline tokens with identity
headers. This changes how the onboarding service creates keys.

---

## Migration Path

### Phase 0 — Parallel run (validate MaaS CRDs match current access)

1. Create `MaaSModelRef` CRDs for all models (already needed for Grid)
2. Create `MaaSSubscription` CRDs matching existing subscription names
   with `tokenRateLimits` equivalent to current `MONTHLY_TOKEN_QUOTA`
3. Create `MaaSAuthPolicy` CRDs matching current `model_access` rules
4. Verify: `GET /v1/models` returns the same models as before
5. Verify: all existing keys still validate (maas-api is unchanged)
6. **No enforcement change yet** — Pricetag's `external_metering`
   still handles rate limiting

### Phase 1 — Enable MaaS enforcement

1. Enable Kuadrant `TokenRateLimitPolicy` enforcement (Limitador)
2. Disable `external_metering` balance check (keep usage recording)
3. Verify: rate limiting works per-subscription per-model
4. Remove `model_access` filter from praxis.yaml (MaaSAuthPolicy
   handles access control)
5. Existing API keys continue to work — subscription binding is
   already in the key

### Phase 2 — Enhance quotas

1. Add multi-window limits to `MaaSSubscription` (hourly + monthly)
2. Add per-model differentiation (expensive models get tighter limits)
3. Migrate onboarding from offline token to OpenShift token / SSO

### Phase 3 — Retire Pricetag-specific components

1. Retire `MONTHLY_TOKEN_QUOTA` env var
2. Retire `model_access` filter config
3. Retire `external_metering` balance check (keep usage recording
   until Praxis `token_rate_limit` migration)
4. metering-service continues for usage recording / billing

---

## CRD Examples for Migration

### Current Pricetag config → MaaS CRDs

**Before** (praxis.yaml):
```yaml
- filter: model_access
  denylist:
    - model: claude-fable-5
      groups: ["*"]
      override_groups: ["octo-eng"]
```

**After** (MaaS CRDs):
```yaml
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSAuthPolicy
metadata:
  name: claude-fable-5-access
spec:
  modelRefs:
    - name: claude-fable-5
  subjects:
    groups:
      - name: octo-eng
---
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSSubscription
metadata:
  name: octo-eng-frontier
spec:
  owner:
    groups:
      - name: octo-eng
  modelRefs:
    - name: claude-fable-5
      tokenRateLimits:
        - limit: 5000000
          window: "720h"
        - limit: 200000
          window: "1h"
  priority: 10
```

**Before** (env var):
```
MONTHLY_TOKEN_QUOTA=5000000
```

**After** (in subscription CRD above):
```yaml
tokenRateLimits:
  - limit: 5000000
    window: "720h"      # ~monthly
```

---

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **Governance pairing removes models** | Models without both subscription + auth policy disappear from API | Phase 0 validation — verify `GET /v1/models` before enabling enforcement |
| **Rate limit mismatch** | MaaS per-model limits stricter/looser than global Pricetag quota | Set initial MaaS limits to match current `MONTHLY_TOKEN_QUOTA` per model, then tune |
| **Key creation auth change** | Offline token + identity headers → OpenShift token | Phase 2 migration — keep offline token path working until SSO is ready |
| **Subscription auto-selection** | MaaS picks highest-priority subscription. User with multiple subscriptions may get unexpected limits | Set clear priorities. Document which subscription each team maps to |
| **Limitador vs external_metering overlap** | Running both simultaneously double-counts | Phase 1: disable external_metering balance check BEFORE enabling Limitador |
| **Usage recording gap** | If external_metering is fully disabled, CloudEvent pipeline breaks | Keep external_metering in record-only mode (no balance check) |

---

---

## The Third Target: Grid-native Token Rate Limiting

MaaS CRDs (`MaaSSubscription.tokenRateLimits`) are not the final
state. The Praxis `token_rate_limit` filter is being built as a
self-contained replacement for Kuadrant/Limitador, with capabilities
that exceed what MaaS CRDs currently express.

**Do not treat MaaS CRDs as the stable long-term target.** They will
need to evolve — or be replaced — to accommodate Grid-level features.

### Three-system comparison

| Capability | Pricetag (current) | MaaS Limitador (interim) | Grid token_rate_limit (target) |
|-----------|-------------------|-------------------------|-------------------------------|
| **Algorithm** | Monthly MTD aggregation | Fixed window | Sliding window + token bucket |
| **Reservation** | No (coarse yes/no) | No (counts after response) | Yes (reserve/reconcile) |
| **Scope** | Per-user, global | Per-user, per-subscription, per-model | Per-subject, per-model, composite keys |
| **Multi-window** | No (monthly only) | Yes (CRD: multiple `tokenRateLimits`) | Yes (multiple rules per filter) |
| **Concurrent limits** | No | No | No (gap across all three) |
| **Hierarchical quotas** | No | No | Planned (org → team → user via composite keys) |
| **Soft/shadow enforcement** | No | No | Planned (#1381) |
| **Pre-forward ceiling** | No | No | Planned (#1332 — reject oversized input) |
| **Token-type weights** | No | No | Shipped (#1132 — input/output/reasoning weighted differently) |
| **Error refund** | No | No | In reserve/reconcile (needs verification) |
| **State backend** | PostgreSQL (query per request) | Limitador (in-memory/Redis) | In-process memory or Valkey |
| **Cross-replica** | Yes (shared DB) | Yes (shared Limitador) | Only with Valkey |
| **Config source** | Env var | CRD (`MaaSSubscription`) → auto-generated TRLP | praxis.yaml / Grid overlay |
| **CRD-driven** | No | Yes | No (needs CRD → config generation) |
| **Usage recording** | Yes (CloudEvents → PostgreSQL) | No | No (critical gap) |
| **Per-app identity** | No | No (per-subscription) | Possible (header-based composite key) |
| **Estimation strategies** | No | No | 4 strategies (fixed, max_tokens, input+max, model_scaled) |
| **In hot path** | HTTP callout (latency) | Limitador sidecar call | Zero (in-process) |

### What MaaS CRDs can't express (but Grid needs)

| Feature | MaaS CRD gap | Grid token_rate_limit |
|---------|-------------|----------------------|
| **Reserve/reconcile** | `TokenRateLimitPolicy` counts after response. No admission reservation | Core design — reserves at admission, reconciles after response |
| **Sliding window** | Fixed window only (limit resets at boundary) | Sliding window — smooth, no boundary spike |
| **Estimation strategy** | No concept — Limitador counts actual usage only | 4 configurable strategies for pre-forward estimation |
| **Soft enforcement** | Hard 429 only | `enforcement: hard|soft|shadow` per rule |
| **Composite keys** | Counter key is `auth.identity.userid` only | Composable: `[subject, model, header, ip]` in any combination |
| **Token-type weights** | Counts `total_tokens` only | Weights input/output/reasoning differently |
| **Cross-cluster enforcement** | Per-Limitador instance (site-local) | Valkey backend shared across clusters |
| **Pre-forward ceiling** | No | Reject requests with oversized input before forwarding |

### The evolution path

```
Phase 0: Pricetag (external_metering + metering-service)
  │  → global env var, balance check, no reservation
  │
Phase 1: MaaS (MaaSSubscription + Limitador)           ← interim
  │  → CRD-driven, per-model, per-user, fixed window
  │  → MaaS CRDs are the CONFIG SOURCE, Limitador is the ENFORCER
  │
Phase 2: Grid (token_rate_limit + Valkey)               ← target
  │  → reserve/reconcile, sliding window, composite keys
  │  → CRD still the CONFIG SOURCE, but CRD schema evolves
  │  → token_rate_limit replaces Limitador as the ENFORCER
  │
Phase 3: Unified CRD (MaaSSubscription evolved or new CRD)
     → CRD expresses: estimation strategy, enforcement mode,
        composite keys, multi-window, token-type weights
     → Controller generates token_rate_limit config (not TRLP)
     → Limitador retired
```

### What this means for migration planning

1. **Don't optimize for MaaS CRD schema stability.** The
   `MaaSSubscription.tokenRateLimits` schema (`limit` + `window`)
   is too simple for the target state. It will need new fields:
   `algorithm`, `estimation`, `enforcement`, `key` (composite),
   `tokenTypeWeights`. Plan for CRD evolution, not CRD stability.

2. **Treat MaaS Limitador as interim enforcer.** It works for Phase 1
   (per-model fixed-window limits) but lacks reservation, sliding
   window, and cross-cluster enforcement. The migration from Limitador
   to `token_rate_limit` is a second migration after the Pricetag →
   MaaS migration. Don't invest heavily in Limitador integration.

3. **Invest in the config generation layer.** The durable value is in
   the CRD → config pipeline (admin creates CRD → controller generates
   enforcement config). Whether the enforcer is Limitador or
   `token_rate_limit`, the CRD is the admin interface. Build the
   controller to be enforcer-agnostic where possible.

4. **Usage recording is orthogonal.** Neither Limitador nor
   `token_rate_limit` records usage for billing. This capability
   (CloudEvents → PostgreSQL) must be preserved independently of
   which enforcer is active. Don't couple recording to enforcement.

5. **The real migration is two hops, not one:**
   ```
   Pricetag → MaaS/Limitador → Grid/token_rate_limit
   ```
   Optimizing the first hop (Pricetag → MaaS) for a CRD schema that
   will change in the second hop (MaaS → Grid) creates rework. Where
   possible, skip directly to Grid-compatible patterns:
   - Use `token_rate_limit` in shadow mode alongside Limitador
   - Design CRD fields for the Grid target, even if Limitador ignores them
   - Keep the enforcement boundary clean (CRD → controller → enforcer)

---

## What Doesn't Migrate (remains Pricetag-specific)

| Component | Reason |
|-----------|--------|
| metering-service | Usage recording, billing, cost dashboards — MaaS doesn't replace these |
| LiteLLM pricing sync | Cost calculation — MaaS has no pricing module |
| Cost center mapping | LDAP integration for chargeback — Pricetag-specific |
| metering dashboard | Replaced by Grafana (Thanos) for operational views, but billing detail views may persist |
