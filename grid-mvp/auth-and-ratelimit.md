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

### Decision: API key auth at MaaS layer only

**MaaS Gateway** (on ingress clusters) handles all authentication and
authorization via Authorino (Kuadrant). **Grid Gateway** does routing
only — no `api_key_auth`, no identity resolution.

```
Consumer
  │  Bearer sk-oai-abc123...
  ▼
MaaS Gateway (Ingress cluster)
  ├── Authorino (Kuadrant)
  │   Validates API key via maas-api
  │   Resolves subscription → selected_subscription_key
  │   Returns: username, groups
  │
  ├── Limitador (Kuadrant)
  │   Token rate limit per user/subscription/model
  │
  ├── Credential injection
  │   For llm-d models: injects Grid service token (or passthrough)
  │   For external APIs: injects provider API key (e.g., ANTHROPIC_API_KEY)
  │
  └── Routes to Grid GW (for llm-d models) or external API directly
        │
        ▼
Grid Gateway (same ingress cluster, routing only)
  ├── model_to_header
  │   Extracts model name from body → X-Gateway-Model-Name
  │
  ├── intelligent_route
  │   Reads overlay candidates, selects target site
  │
  └── load_balancer → forwards to target standalone GW
        (local cluster or remote via mTLS)
```

### Auth happens once — at MaaS

| Concern | MaaS Gateway (Authorino) | Grid Gateway |
|---------|--------------------------|-------------|
| **What it checks** | Is the key valid? Does this user's subscription allow this model? | Is the request from a trusted service? (SA token validation) |
| **What it returns** | Subscription key for rate limiting, identity for metering | N/A — forwards to target |
| **Scope** | User authentication, authorization, rate limiting, metering | Service-level auth + cross-cluster routing |
| **Auth mechanism** | Kuadrant (Authorino + Limitador) | K8s ServiceAccount tokens (same-cluster), Grid mTLS (cross-cluster) |

Grid GW authenticates requests at the service level, not the user
level. No user API key validation — MaaS handles that upstream.

- **Same cluster** (MaaS GW → Grid GW, Grid GW → llm-d GW):
  ServiceAccount token validation. The sender pod mounts a projected
  SA token; the receiver validates via K8s TokenReview.
- **Cross-cluster** (Grid GW → Grid GW): mTLS between Grid GWs at
  different sites (existing Grid certificate infrastructure).

### Filter chain: Grid Gateway (MVP)

```yaml
filter_chains:
  - name: grid-route
    filters:
      - filter: model_to_header      # body → X-Gateway-Model-Name
      - filter: intelligent_route    # overlay candidates, site selection
        local_site: <this-site>
      - filter: load_balancer        # → selected target standalone GW
```

No `api_key_auth` — MaaS GW handles all auth upstream.
No `token_rate_limit` — MaaS handles limits via Limitador.
No `match_claims` on `X-Grid-Region` — geo fencing is deferred to
Phase 2, driven by MaaS GW headers when tenant region metadata is
available via Authorino CRD configuration.

### Filter chain: MaaS Gateway (entry point)

The MaaS Gateway runs its existing Kuadrant-based pipeline as the
customer entry point on ingress clusters. It validates the API key,
enforces rate limits, injects credentials, and forwards to Grid GW
(for llm-d models) or directly to external APIs.

### maas-api validation response: current vs extended

The primary consumer of the extended response is MaaS GW itself
(for geo fencing once enabled in Phase 2). Grid GW does not call
maas-api — it does routing only.

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

**Extended response** (Phase 2):
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

New fields (Phase 2):
- `region` — from `AITenant.spec.region` or `MaasTenantConfig`.
  Consumed by MaaS GW to set `X-Grid-Region` header before
  forwarding to Grid GW.
- `tokenBudget` — from `MaaSSubscription.tokenRateLimits` for the
  matched subscription. Consumed by MaaS GW for budget visibility.

### AuthenticatedIdentity bridge

Grid GW does not need the Praxis `AuthenticatedIdentity` extension.
It does routing only — no identity, roles, or claims are required.
The `intelligent_route` filter selects candidates based on model
name and overlay scoring, not identity-based filtering.

**Post-MVP**: If Grid-level `token_rate_limit` is enabled, the
identity bridge becomes relevant. At that point, MaaS GW
would pass identity via a trusted header (e.g., signed
`X-Authenticated-Subject`) that Grid GW reads into
`AuthenticatedIdentity` for per-user bucketing.

---

## Management Layer Auth: Admin vs User

The data plane auth handles inference requests. But the management
plane — model listing, key creation, usage queries — also needs auth.
The two personas have different requirements:

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
`POST /v1/api-keys`, `GET /v1/subscriptions`), requiring an OpenShift account is
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
| **Stack** | Praxis AI + Kuadrant + Authorino + Limitador (migrating from Envoy) | Self-contained in Praxis |

### MVP: MaaS only

```
Consumer → MaaS Gateway → Limitador checks budget
                         → if OK: forward to Grid GW → routing → inference
                         → if over: HTTP 429 (never reaches Grid GW)
```

- `token_rate_limit` disabled at Grid Gateway
- Kuadrant/Limitador enforces per-user, per-model limits at MaaS GW
  (ingress clusters)
- Rate-limited requests never reach Grid — reduces unnecessary
  cross-cluster traffic
- No cross-site budget enforcement (deferred)
- Config driven by `MaaSSubscription.tokenRateLimits` CRD

### Post-MVP: Complementary layers

```
Consumer → MaaS Gateway → Limitador (per-model budget)
                         → if over model budget: HTTP 429
                         → Grid Gateway → token_rate_limit (global per-subject budget, Valkey)
                                        → if over global: HTTP 429
                                        → routing → inference
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
