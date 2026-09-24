# AI Grid — Auth and Rate Limiting Flows

Detailed comparison and design for authentication and rate limiting
across the Grid and MaaS layers.

---

## Auth: Two layers, different concerns

### Current state: v3 demo vs dogfood

| | Dogfood (MaaS, production) | v3 Demo (Grid) |
|--|---------------------------|----------------|
| **Credential** | API key (`x-api-key` / `Bearer sk-oai-...`) | JWT (`Bearer eyJ...`) |
| **Validation** | External call to maas-api | Local HS256 signature check |
| **Identity resolution** | maas-api returns user/group | JWT claims (`sub`) |
| **Geo/budget claims** | Not in the key | Embedded in JWT (`grid_region`, `grid_rate`) |
| **DB dependency** | Yes (key hash lookup) | No |
| **Filter** | `api_key_auth` | `policy` (PPE `user-jwt` plugin) |

These are fundamentally different auth models. The v3 demo sidesteps
MaaS auth entirely.

### Decision: API key auth at both layers

**Grid Gateway** uses `api_key_auth` → maas-api (same as today's dogfood).
**MaaS Gateway** uses Authorino (Kuadrant) for subscription-level authorization.

Both layers are in the request path:

```
Consumer
  │  Bearer sk-oai-abc123...
  ▼
Grid Gateway
  ├── api_key_auth → POST /internal/v1/api-keys/validate → maas-api
  │   Returns: username, groups, region, token_budget
  │   Sets: AuthenticatedIdentity (subject_id = username)
  │         X-Grid-Region header (from tenant config)
  │
  ├── intelligent_route
  │   Reads X-Grid-Region → matches against candidate labels
  │   Selects target site
  │
  └── load_balancer → forwards to target site's MaaS Gateway
        │
        ▼
MaaS Gateway
  ├── Authorino (Kuadrant)
  │   Validates API key again (MaaSAuthPolicy enforcement)
  │   Resolves subscription → selected_subscription_key
  │
  ├── Limitador (Kuadrant)
  │   Token rate limit per user/subscription/model
  │
  ├── Credential injection
  │   Injects provider API key (e.g., ANTHROPIC_API_KEY)
  │
  └── Routes to backend (EPP → vLLM or ExternalModel → external API)
```

### Auth happens twice — by design

| Concern | Grid Gateway (api_key_auth) | MaaS Gateway (Authorino) |
|---------|---------------------------|--------------------------|
| **What it checks** | Is the key valid? Who is this user? | Does this user's subscription allow this model? |
| **What it returns** | Identity + region + budget | Subscription key for rate limiting |
| **Scope** | Grid-level routing decisions | Site-level access policy |
| **Could be skipped?** | No — needed for routing | Potentially, if Grid passes trusted identity headers |

**Future optimization**: Grid Gateway could pass a signed identity
header that MaaS Gateway trusts, skipping the second maas-api call.
Not needed for MVP — the overhead is small (maas-api is in-cluster).

### Filter chain: Grid Gateway (MVP)

```yaml
filter_chains:
  - name: grid
    filters:
      - filter: api_key_auth         # → maas-api validate
      - filter: model_to_header      # body → X-Gateway-Model-Name
      - filter: intelligent_route    # overlay candidates, X-Grid-Region
        local_site: hub
        match_claims:
          - { claim_header: X-Grid-Region, label: region }
      - filter: load_balancer        # → selected site's MaaS Gateway
```

Note: `token_rate_limit` is **not** in the Grid Gateway chain for MVP.
Rate limiting is MaaS-only.

### Filter chain: MaaS Gateway (existing, unchanged)

The MaaS Gateway runs its existing Kuadrant-based pipeline. The
Grid Gateway's request arrives as a proxied HTTP request with the
original API key still in the Authorization header. Authorino
validates it independently.

### maas-api validation response: current vs extended

**Current response** (from `/internal/v1/api-keys/validate`):
```json
{
  "valid": true,
  "username": "alice@example.com",
  "groups": ["team-a", "platform"],
  "subscription": "team-a-sub",
  "keyId": "550e8400-e29b-..."
}
```

**Extended response** (to build):
```json
{
  "valid": true,
  "username": "alice@example.com",
  "groups": ["team-a", "platform"],
  "subscription": "team-a-sub",
  "keyId": "550e8400-e29b-...",
  "region": "us-east-1",
  "tokenBudget": {
    "limit": 1000000,
    "window": "720h",
    "remaining": 842150
  }
}
```

New fields:
- `region` — from `AITenant.spec.region` or `MaasTenantConfig`
- `tokenBudget` — from `MaaSSubscription.tokenRateLimits` for the
  matched subscription

### AuthenticatedIdentity bridge

The `api_key_auth` filter must populate the Praxis `AuthenticatedIdentity`
extension so downstream filters can read it:

```
AuthenticatedIdentity {
  subject_id: "alice@example.com"     // from validation response
  roles: {"team-a", "platform"}       // from groups
  custom_claims: {
    "region": "us-east-1",            // from extended response
    "subscription": "team-a-sub"
  }
}
```

The `token_rate_limit` filter (when enabled post-MVP) reads
`subject_id` for per-user bucketing. The `intelligent_route` filter
reads the region from a header set by `api_key_auth`.

---

## Management Layer Auth: Admin vs User

The data plane auth (api_key_auth → maas-api) handles inference
requests. But the management plane — model listing, key creation,
usage queries — also needs auth. The two personas have different
requirements:

| | Tenant Admin | Tenant User |
|--|-------------|-------------|
| **Has OpenShift account?** | Yes — platform ops | **Not necessarily** — external engineer |
| **Acceptable auth** | OpenShift token (TokenReview), K8s RBAC | API key, SSO/OIDC, corporate IdP |
| **NOT acceptable** | — | Requiring an OpenShift login |
| **Accesses** | Admin UI → Management API → ACM | User Portal → maas-api |

### The gap

Today maas-api supports two auth modes:
1. **OpenShift token** — `TokenReview` + `SubjectAccessReview` (strict auth)
2. **API key** — validated via the same `/internal/v1/api-keys/validate` path

For tenant users accessing self-service endpoints (`GET /v1/models`,
`POST /v1/api-keys`, usage/quota), requiring an OpenShift account is
not viable — 8K+ engineers won't all have cluster access.

### Options

1. **API key for management too**: The user's existing API key
   authenticates them for self-service endpoints. maas-api already
   resolves key → username. Chicken-and-egg: need a key to create a
   key (first key must come from admin or onboarding flow).

2. **SSO/OIDC (corporate IdP)**: User logs into the portal via
   corporate SSO (e.g., Red Hat SSO, Keycloak). maas-api validates
   the OIDC token. No OpenShift dependency. Portal handles the
   OAuth flow.

3. **Hybrid**: First key created via onboarding (admin-initiated or
   SSO-authenticated). Subsequent self-service uses the API key.

**Recommendation**: Option 2 (SSO/OIDC) for portal access, with
Option 3 for API-only access (scripts, automation). The portal does
the OAuth dance, maas-api validates the OIDC token. For programmatic
access without a browser, users use their API key.

---

## Rate Limiting: MaaS (MVP) vs Grid (post-MVP)

### Side-by-side comparison

| Aspect | MaaS / Limitador | Grid / Praxis `token_rate_limit` |
|--------|-----------------|--------------------------------|
| **Counts** | LLM tokens (`usage.total_tokens` from response) | LLM tokens (from `token_count` filter) |
| **Scope** | Per-user, per-subscription, **per-model** | Per-subject, **across all models** |
| **Algorithm** | Fixed window | Sliding window or token bucket |
| **Runs where** | External Limitador deployment + Envoy wasm-shim | In-process in Praxis binary |
| **State** | Limitador (in-memory or Redis) | In-process memory or Valkey |
| **Config** | CRD-driven (MaaSSubscription → auto-generated TRLP) | Config file (praxis.yaml) or overlay |
| **Multi-replica** | Shared via Limitador | Shared only with Valkey backend |
| **Estimation** | None — counts after response | Reserve/reconcile: estimates upfront |
| **API formats** | OpenAI only. **Anthropic NOT supported** | OpenAI (provider-configurable) |
| **Stack** | Istio/Envoy + Kuadrant + Authorino + Limitador | Self-contained in Praxis |

### MVP: MaaS only

```
Consumer → Grid Gateway (NO rate limiting) → MaaS Gateway → Limitador checks budget
                                                           → if OK: inference
                                                           → if over: HTTP 429
```

- `token_rate_limit` disabled at Grid Gateway
- Kuadrant/Limitador enforces per-user, per-model limits at each site
- No cross-site budget enforcement (deferred)
- Config driven by `MaaSSubscription.tokenRateLimits` CRD

### Post-MVP: Complementary layers

```
Consumer → Grid Gateway → token_rate_limit (global per-subject budget, Valkey)
                        → if over global: HTTP 429
                        → MaaS Gateway → Limitador (per-model budget)
                                       → if over model budget: HTTP 429
                                       → inference
```

Two non-overlapping scopes:
- **Grid**: "alice can use 500K tokens/day total across all sites and all models"
- **MaaS**: "team-a can use 1M tokens/month on claude-sonnet on this site"

### Double-counting prevention

Only one layer deducts tokens at a time:
- **MVP**: MaaS only — no risk
- **Post-MVP**: Grid deducts from global budget, MaaS deducts from
  per-model budget. These are independent counters. A request that
  passes Grid's global check can still be rejected by MaaS's model
  check (and vice versa). No double-counting because they measure
  different things.

### Limitador gap: Anthropic format

MaaS/Limitador reads `usage.total_tokens` from the response body.
This field exists in OpenAI Chat Completions responses but NOT in
Anthropic Messages responses (which use `input_tokens` /
`output_tokens`). This means:

- **Rate-limited today**: OpenAI models, OpenAI-compatible (vLLM)
- **NOT rate-limited**: Anthropic models via Anthropic-native format

Praxis AI's `token_count` filter is provider-aware and handles both
formats. This is one reason Grid-level rate limiting is the long-term
target — it works with all provider formats.

### Rate limiting configuration flow

```
Tenant Admin
  └── creates MaaSSubscription with tokenRateLimits:
        modelRefs:
          - name: claude-sonnet
            tokenRateLimits:
              - limit: 100000
                window: "1h"
              - limit: 1000000
                window: "720h"
      │
      └── maas-controller reconciles
            └── generates TokenRateLimitPolicy (Kuadrant CRD)
                  targetRef: HTTPRoute for claude-sonnet
                  counters: auth.identity.userid
                  rates: [{limit: 100000, window: "1h"}, ...]
                  when: auth.identity.selected_subscription_key == "..."
      │
      └── Limitador enforces at runtime
            └── reads usage.total_tokens from response
            └── increments per-user counter
            └── returns 429 when budget exhausted
```

### Open design question (ai#127)

The Praxis `token_rate_limit` module explicitly notes:

> "open questions remain: HA/clustered-Valkey failure modes, and this
> filter's relationship to Kuadrant's `TokenRateLimitPolicy`"

This confirms the Praxis team is aware of the overlap and hasn't
resolved the boundary. Our MVP decision (MaaS only) lets this
question settle before taking a dependency on either approach.
