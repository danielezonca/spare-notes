# Rate Limiting: external_metering vs token_rate_limit

Comparison of the Pricetag dogfood's current rate limiting (external_metering
and metering-service) vs the Praxis native token_rate_limit filter, and the
change needed to adopt it.

---

## Feature Comparison

| Feature | external_metering (current) | token_rate_limit (target) |
|---------|---------------------------|--------------------------|
| **Architecture** | External HTTP callout per request to metering-service | In-process filter, no external call |
| **State storage** | PostgreSQL (metering_events table) | In-process memory or Valkey (Redis) |
| **Latency impact** | HTTP roundtrip per request (mitigated by timeout + fail_open) | Zero (in-process). Async Valkey reconciliation if using shared state |
| **Budget check** | Pre-request: coarse yes/no (`hasAccess`) based on MTD aggregation | Pre-request: reserve estimated tokens upfront, reject if over budget |
| **Post-response** | CloudEvent with actual token counts → metering-service records in DB | Reconcile: refund or charge overage based on actual usage from `token_count` |
| **Reservation** | No — race condition possible (burst of concurrent requests all pass the check before any usage is recorded) | Yes — tokens reserved at admission time, concurrent requests compete for remaining capacity |
| **Quota scope** | Per-user, single global env var (`MONTHLY_TOKEN_QUOTA`) | Per-subject, configurable per rule. Multiple rules supported |
| **Per-model limits** | No — single budget across all models | Possible with multiple rules keyed differently |
| **Algorithm** | Cumulative usage vs fixed monthly cap | Sliding window or token bucket (configurable per rule) |
| **Window** | Monthly only (MTD aggregation query) | Any duration (1m, 1h, 24h, 720h, etc.) |
| **Multi-replica** | Consistent (shared PostgreSQL) | Memory: per-replica (inaccurate). Valkey: shared (accurate) |
| **Estimation strategies** | None — yes/no check only | `fixed`, `max_tokens`, `input_plus_max_tokens`, `model_scaled` |
| **Fail mode** | Configurable: `fail_open: true` (allow on metering-service failure) | Configurable per backend |
| **Concurrent request limiting** | No | No (neither supports this) |
| **Provider format** | Relies on `token_count` filter output (provider-aware) | Same — reads from `token_count` filter metadata |
| **Multi-tier quotas** | No — single monthly cap only | Yes — multiple rules (hourly + daily + monthly) evaluated per request, most restrictive wins |
| **Config source** | Env var + metering-service DB | praxis.yaml config file (or Grid overlay) |
| **CRD-driven** | No — manual env var | No — but could be generated from CRDs by a controller |
| **Usage recording** | Yes — CloudEvents → PostgreSQL (reporting, billing, chargeback) | **No** — only enforces limits, does not persist usage history. **Critical gap for billing** |
| **Dashboard / reporting** | Yes — metering-service serves usage dashboards | No — separate observability needed (Prometheus/Grafana) |
| **Cost awareness** | Partial — pricing module exists but entitlement uses raw token counts | No — token counts only, no dollar amounts |
| **Dependencies** | metering-service (Go), PostgreSQL, CloudEvents | None (in-process). Optional: Valkey for multi-replica |

## Key Differences

### 1. Reservation prevents overshooting

The biggest architectural difference. With `external_metering`, a burst
of 10 concurrent requests all check the balance simultaneously — all see
"has budget" — all proceed — actual usage may exceed the quota by 10x
before the next check sees the updated balance.

`token_rate_limit` reserves tokens at admission time. If the budget is
1000 tokens and each request reserves 100, the 11th request is rejected
immediately. After the response, the actual usage is reconciled (refund
if estimate was high, charge if low).

### 2. No external call = no latency tax

`external_metering` adds an HTTP roundtrip to every inference request.
Even with timeouts and fail_open, this is measurable latency on every
request. `token_rate_limit` is in-process — zero additional latency.

### 3. Usage recording MUST NOT be lost (critical requirement)

`external_metering` records every request's token usage in PostgreSQL
via CloudEvents. This powers:
- **Reporting**: per-user, per-model, per-team usage dashboards
- **Billing**: cost reconciliation with provider invoices
- **Chargeback**: internal cost allocation to teams/cost centers

`token_rate_limit` only enforces limits — it does not persist usage
data anywhere. Prometheus metrics provide aggregated counters but
not the per-request granularity needed for billing line items.

**This is a hard requirement**: any change MUST preserve
per-request usage recording with enough detail for billing and
chargeback (user, model, tokens, timestamp, cost). Prometheus/Thanos
is sufficient for dashboards but NOT for billing reconciliation —
billing needs transactional records, not aggregated time series.

**The usage recording pipeline must survive regardless
of which enforcement mechanism is used.** This means either:
- Keep a usage-recording filter (CloudEvents or similar) in the
  chain alongside `token_rate_limit`
- Or build an equivalent recording mechanism into `token_rate_limit`
  itself (emit events on reconciliation)

### 4. Window flexibility and multi-tier quotas

`external_metering` only supports a single monthly budget (MTD
aggregation against one `MONTHLY_TOKEN_QUOTA` value). Real-world
quota requirements are more complex:

- **Daily burst cap**: prevent one user from consuming the entire
  month's budget in a single day
- **Monthly aggregate**: total consumption ceiling for billing
- **Per-model limits**: different budgets for expensive vs cheap models
- **Hourly spike protection**: prevent runaway loops from draining budget

`token_rate_limit` supports all of these simultaneously via multiple
rules — each with its own window, capacity, and algorithm:

```yaml
- filter: token_rate_limit
  key: authenticated_subject
  rules:
    - name: hourly_spike
      algorithm: sliding_window
      window: 1h
      capacity: 50000
    - name: daily_burst
      algorithm: sliding_window
      window: 24h
      capacity: 200000
    - name: monthly_cap
      algorithm: sliding_window
      window: 720h
      capacity: 5000000
```

All rules are evaluated per request. The most restrictive one wins —
if the hourly budget is exhausted, the request is rejected even if the
monthly budget has capacity. This is a significant upgrade over the
single monthly check in `external_metering`.

### 5. Multi-replica accuracy

`external_metering` is naturally consistent across replicas because
it queries PostgreSQL. `token_rate_limit` with in-memory state is
per-replica (2 replicas = effectively 2x the budget). Valkey backend
is needed for multi-replica accuracy — adds an infrastructure
dependency but no per-request latency (async reconciliation).

---

## The Change: Split Enforcement from Recording

```
Filter chain:
  api_key_auth → ... → token_rate_limit → external_metering → token_count → ...
                         ↑                    ↑
                    enforcement          usage recording only
                  (reserve/reconcile)    (disable balance check)
```

- `token_rate_limit`: handles enforcement (reserve, reject, reconcile)
  with multi-tier rules (hourly spike + daily burst + monthly cap)
- `external_metering`: demoted to usage recorder only — sends CloudEvents
  post-response but skips the pre-request balance check
- **Billing/chargeback pipeline preserved**: CloudEvents → PostgreSQL →
  reporting, billing reconciliation, cost allocation
- Dashboard and admin queries continue to work unchanged
- Requires a config flag on `external_metering` to disable the balance
  check while keeping the event reporting

### What this delivers

This single change delivers the full enforcement upgrade with zero disruption
to the recording pipeline:

- Reserve/reconcile (no overshooting)
- Sliding window (no boundary spikes)
- Multi-tier quotas (hourly + daily + monthly)
- Anthropic format support
- Hot-path HTTP call eliminated
- Limitador retired
- Usage recording preserved (CloudEvents → PostgreSQL unchanged)
- Billing/chargeback pipeline intact

CloudEvent recording is already async (`tokio::spawn`, fire-and-forget)
— no optimization needed. Grafana replacing dashboards happens
independently via Grid/ACM. Further optimizations (retiring
metering-service entirely) are optional future work — see end of doc.

---

## Prerequisites

### Critical (blockers)

| Prerequisite | Description |
|-------------|-------------|
| `external_metering` record-only mode | Config flag to disable pre-request balance check while keeping post-response CloudEvent recording. Usage recording pipeline MUST be preserved for billing/chargeback |
| `token_rate_limit` multi-rule support | Multiple rules per filter instance (hourly + daily + monthly). Already supported in code — needs config validation |
| Per-subject rules | Configurable per-subject budgets (not global env var). Already supported via `key: authenticated_subject` |

### Important (when scaling)

| Prerequisite | Description |
|-------------|-------------|
| `token_rate_limit` with Valkey | Multi-replica state sharing for accurate enforcement. Without it, 2 replicas = effectively 2x budget. Needed when running multiple gateway replicas |
| CRD-driven config generation | Generate `token_rate_limit` rules from MaaS subscription or similar CRD source. Without it, config is manual praxis.yaml |

---

## Precondition: Cost-based budget enforcement

**This must be resolved before adopting `token_rate_limit`.**

Pricetag currently enforces "$300/month" budgets. `token_rate_limit`
counts **tokens**, not **dollars**. Different models have wildly
different per-token costs (Claude Sonnet input: $3/MTok, output:
$15/MTok vs GPT-4o-mini: $0.15/$0.60). Switching from
`external_metering` to `token_rate_limit` without solving this
**loses the dollar-based enforcement that Pricetag provides today**.

**What Pricetag has today**: metering-service pricing module
(`internal/pricing/pricing.go`) with LiteLLM pricing data. Cost is
calculated per-request from token counts × model pricing. The balance
check enforces a monthly dollar cap via token-to-cost conversion.

**Options**:
1. **Token budgets per model** — set model-specific token limits that
   approximate the dollar budget (e.g., 500K tokens/month on Claude
   Sonnet ≈ $300). Simple, requires admin to do the math. Breaks
   when pricing changes.
2. **Cost-weighted tokens** — use `token_rate_limit`'s token-type
   weights (#1132) to normalize tokens to a cost unit per model.
   Budget is in "cost units" not raw tokens.
3. **Cost calculation in estimation strategy** — extend
   `token_rate_limit` estimation to multiply tokens × price at
   reservation time. Budget expressed in cents. Upstream feature
   request.
4. **Keep `external_metering` balance check for dollar budgets** —
   don't disable the balance check for dollar-denominated budgets.
   Use `token_rate_limit` for token budgets (burst/daily/hourly) and
   `external_metering` for the monthly dollar cap. Two enforcers
   with non-overlapping scopes.

**Recommendation**: Option 4 until a Praxis-native cost-aware
enforcement exists — keep `external_metering` for the monthly dollar
cap, use `token_rate_limit` for token-based burst/daily limits. This
preserves Pricetag's current dollar enforcement while adding the new
capabilities. The change to `token_rate_limit` is additive, not a
replacement, until cost-based enforcement is available natively.

## Tracking: Concurrent request limiting

**Not a regression for Pricetag** — MaaS already provides this via
Kuadrant/Limitador request-rate policies. Ongoing work in Praxis to
include it natively. Track but no action needed for this change.

**Current state**:
- MaaS/Limitador: can enforce request-count limits per subscription
  (not token-count). Available today via `RateLimitPolicy` CRD.
- Praxis: `rate_limit` filter does request-count rate limiting
  (token bucket). Available in core. Does requests/second, not
  concurrent in-flight.
- Praxis `token_rate_limit`: token-count only, no request-count or
  concurrency. Compositional keys (#1334) may enable per-app keying.
- Customer requirement: max N simultaneous requests per app+model.
  This is a concurrency (semaphore) concern, distinct from rate
  (requests/second) or budget (tokens/window).

**Tracking**: Praxis team is aware. No specific issue for
semaphore-style concurrency limiting yet. MaaS/Limitador covers the
request-rate case for now. Flag if customer escalates concurrency
as a blocker.

---

## Optional Future Work — Retire metering-service from data path

If metering-service becomes an operational burden or the recording
pipeline needs to change, the `external_metering` filter could be
replaced entirely. This is NOT needed and provides
marginal benefit (one less Go service to operate) at significant cost
(building a recording replacement).

**Options if pursued:**
- **Option A**: Keep `external_metering` filter as recorder only
  (current state after the change — already works)
- **Option B**: New lightweight event-emitter filter that writes
  usage events without any enforcement logic
- **Option C**: Extend `token_rate_limit` to emit usage events on
  reconciliation (upstream feature request to Praxis team)

**What would be retired:**
- metering-service balance-check endpoint (already dead after the change)
- metering-service event ingestion (replaced by Option B or C)
- metering-service dashboard (replaced by Grafana/Thanos)

**What must be preserved regardless:**
- Per-request transactional usage records for billing/chargeback
- Token breakdown (prompt, completion, cache, reasoning)
- Identity attribution (user, group, subscription, model, provider)
- CloudEvents format compatibility (if downstream billing systems
  consume it)
