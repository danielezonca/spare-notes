# Rate Limiting Migration: external_metering → token_rate_limit

Comparison of the Pricetag dogfood's current rate limiting (external_metering
+ metering-service) vs the Praxis native token_rate_limit filter, to plan
the migration path.

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

**This is a hard requirement**: any migration plan MUST preserve
per-request usage recording with enough detail for billing and
chargeback (user, model, tokens, timestamp, cost). Prometheus/Thanos
is sufficient for dashboards but NOT for billing reconciliation —
billing needs transactional records, not aggregated time series.

**The usage recording pipeline must survive the migration regardless
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

## Migration Path

### Phase 1 — Dual mode (split enforcement from recording)

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

### Phase 2 — Optimize recording

- Evaluate whether CloudEvent recording can move to an async/buffered
  pattern (batch writes, reduce per-request DB pressure)
- Grafana dashboards (Thanos) replace metering-service dashboard views
  for operational monitoring
- metering-service continues for **billing/chargeback** data (per-request
  transactional records that Prometheus cannot replace)
- Consider splitting metering-service: retire the balance-check endpoint,
  keep the event ingestion + reporting endpoints

### Phase 3 — Full native

- `token_rate_limit` with Valkey backend for multi-replica accuracy
- Multiple rules for hourly/daily/monthly caps
- Prometheus/Thanos/Grafana for operational dashboards
- Usage recording pipeline for billing/chargeback:
  - Option A: `external_metering` filter (event recording only, no
    balance check) + metering-service event ingestion
  - Option B: New lightweight event-emitter filter + event store
  - Option C: `token_rate_limit` extended to emit usage events on
    reconciliation (upstream feature request)
- **metering-service balance-check endpoint retired** — enforcement is
  fully `token_rate_limit`
- **metering-service event ingestion may persist** — depends on
  billing/chargeback requirements

---

## What needs to exist before migration

### Critical (Phase 1 blockers)

| Prerequisite | Description |
|-------------|-------------|
| `external_metering` record-only mode | Config flag to disable pre-request balance check while keeping post-response CloudEvent recording. Usage recording pipeline MUST be preserved for billing/chargeback |
| `token_rate_limit` multi-rule support | Multiple rules per filter instance (hourly + daily + monthly). Already supported in code — needs config validation |
| Per-subject rules | Configurable per-subject budgets (not global env var). Already supported via `key: authenticated_subject` |

### Important (Phase 2-3)

| Prerequisite | Description |
|-------------|-------------|
| `token_rate_limit` with Valkey | Multi-replica state sharing for accurate enforcement across gateway replicas |
| Dashboard migration | Grafana dashboards (Thanos) for operational monitoring. Does NOT replace billing data |
| CRD-driven config generation | Generate `token_rate_limit` rules from MaaS subscription or similar CRD source |
| Billing data strategy | Determine long-term home for per-request transactional usage records (billing, chargeback, cost allocation). Options: keep metering-service event ingestion, new event store, or extend `token_rate_limit` to emit events |

---

## Open Questions

- **Cost-based budgets**: Metering-service has a pricing module
  (`internal/pricing/pricing.go`) for dollar-based budgets. `token_rate_limit`
  only counts tokens. If the platform needs "$300/month" budgets (not
  "5M tokens/month"), cost calculation needs to happen somewhere —
  either in the estimation strategy or as a separate concern.

- **Billing reconciliation**: If external billing systems
  scrape usage data for billing, they currently read from the
  metering-service DB. After migration, they'd need to read from
  Prometheus/Thanos or a new data source. This is an integration
  dependency outside the gateway.

- **Concurrent request limiting**: Neither approach handles "max N
  simultaneous requests." If this is a requirement, it needs a separate
  mechanism (semaphore filter, connection limit at LB level).
