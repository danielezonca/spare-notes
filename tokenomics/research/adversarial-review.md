# Adversarial Review — Requirements Gap Analysis

Cross-reference of customer requirements (59), Pricetag dogfood
requirements (98), design doc inventory (70+), and implementation
inventory (75) to find uncaptured or unaddressed gaps.

---

## Summary

| Category | Total Requirements | Captured in docs | Implemented/planned | **Uncaptured gaps** |
|----------|-------------------|-----------------|--------------------|--------------------|
| Rate limiting | 16 customer + 12 Pricetag | 11 | 10 | **7** |
| Metering / billing | 8 customer + 16 Pricetag | 5 | 8 | **6** |
| Quota management | 12 customer + 7 Pricetag | 4 | 3 | **8** |
| Auth | 3 customer + 6 Pricetag | 4 | 6 | **2** |
| Routing | 12 customer + 10 Pricetag | 4 | 10 | **3** |
| Observability | 6 customer + 6 Pricetag | 2 | 4 | **4** |
| Infra / compliance | 2 customer + 18 Pricetag | 3 | 2 | **8** |

**Total uncaptured gaps: 38** (requirements that exist in customer/Pricetag
sources but are NOT in our design docs or explicitly scoped out).

---

## Critical Gaps — Not captured, customer expects them

### 1. Concurrent request limits (CR-02, CR-37)

**Requirement**: Max simultaneous concurrent requests per app+model pair.
Customer currently enforces this. Neither Praxis `token_rate_limit`,
MaaS/Limitador, nor `external_metering` supports this.

**Impact**: High — customer treats this as a current requirement, not
future. Without it, a single app can monopolize GPU inference capacity.

**Status**: Missing everywhere. Not in design docs, not implemented,
not planned in any Praxis issue.

### 2. Hierarchical quotas: org → team → user (CR-13, CR-26)

**Requirement**: Cascading limits at org > team > user levels. Each
level enforced independently. User within personal quota can still
be blocked by exhausted org-level bucket.

**Impact**: High — needed for enterprise multi-tenant governance.
Praxis Epic #121 mentions hierarchical quotas but no issue tracks
implementation. Our design docs don't capture this.

**Status**: Mentioned in Praxis epic, no implementation, not in our docs.

### 3. Pre-charge with async refund and error handling (CR-03, CR-04, CR-05)

**Requirement**: Pre-charge at ingress using input tokens + max output.
Async refund after completion. Full refund on system errors.

**Impact**: High — this IS Praxis's reserve/reconcile model, but
**error refunds** are not explicitly covered. If the provider errors
after reservation, does the estimate stay charged or get refunded?

**Status**: Reserve/reconcile shipped (experimental). Error refund
behavior not documented or verified. Not in our design docs.

### 4. 429 response with retry-after and refill details (CR-08)

**Requirement**: Standard 429 with `Retry-After`, `X-RateLimit-Limit-Tokens`,
`X-RateLimit-Remaining-Tokens`, `X-RateLimit-Reset` headers.

**Impact**: Medium — needed for well-behaved clients and SDKs.
Praxis Epic #121 specifies this. `token_rate_limit` may already
emit some of these headers, but customer expects the full set.

**Status**: Specified in Praxis epic. Not verified if current
implementation covers all headers. Not in our design docs.

### 5. Quota warning thresholds at 80%/90% (PT-36)

**Requirement**: Alert users via response header when approaching
quota limits (80% and 90% of monthly budget).

**Impact**: Medium — prevents surprise 429s. Users need advance
notice to adjust usage patterns.

**Status**: Not in design docs. Not implemented. Could be done via
`token_rate_limit` response headers or `external_metering` response
enrichment.

### 6. Per-app vs per-user identity distinction (CR-43, CR-59)

**Requirement**: Application identity (project ID) is distinct from
user identity. Rate limits can be keyed on either independently.
App-level concurrency ≠ user-level token budget.

**Impact**: High — customer's current model is app+model keying.
MaaS identity model doesn't distinguish app from user. Praxis
compositional keys (#1334) could address this with header-based
app identity, but the MaaS identity gap is real.

**Status**: Confirmed gap in additional_requirements.md (#7).
Internal tracking in progress. Not in our design docs.

### 7. GPU capacity-based rate limits (CR-17, CR-29)

**Requirement**: Rate limits calculated from actual GPU capacity
(replica counts, tensor parallelism, throughput, latency bounds).
Limits just below hardware saturation.

**Impact**: Medium — meaningful quotas require knowing the capacity
they protect. Without this, quotas are arbitrary numbers.

**Status**: Grid scoring knows about capacity (queue depth, KV cache)
but doesn't feed this into quota calculation. Not in our design docs.

---

## Important Gaps — Not captured, needed for production

### 8. Token type differentiation in metering (CR-19)

**Requirement**: Track cache writes, cache reads, prompt inputs, and
completion outputs separately (not just total_tokens).

**Status**: Praxis `token_count` extracts these. Praxis `token_rate_limit`
has token-type weights (#1132). Pricetag metering-service records some
of these. But our design docs don't specify which token types must be
preserved through the migration.

### 9. Onboarding capacity simulation (CR-10, CR-11, CR-12)

**Requirement**: Collect P50/P99 token distributions, concurrency needs,
peak usage during onboarding. Simulate in UAT. Set budgets with 25% buffer.

**Status**: Not in any doc or implementation. This is an operational
process, not a technical feature — but it needs tooling support.

### 10. Guardrails on max output tokens at onboarding (CR-18)

**Requirement**: Prevent teams from specifying unrealistic max output
token values that starve shared GPU capacity.

**Status**: Pre-forward token ceiling (#1332) addresses runtime
enforcement. But the onboarding guardrail (admin-configured max)
is not captured.

### 11. Key rotation procedure (PT-14)

**Requirement**: Documented process for rotating provider API keys
and user PriceTag API keys.

**Status**: Not in design docs. maas-api has key revocation but no
rotation workflow (create new → migrate → revoke old).

### 12. Service account keys for CI/CD (PT-26)

**Requirement**: Service account key type with configurable or no
quota for CI/CD pipelines (not counted against user budgets).

**Status**: MaaS has ephemeral keys but not a "service account"
key type. Not in our design docs.

### 13. GLM free tier fallback (PT-35)

**Requirement**: When frontier model quota is exhausted, route to
free on-premise model (GLM 5.3) instead of returning 429.

**Status**: Not captured. Would need soft enforcement (#1381) +
intelligent routing to redirect to free tier. Interesting use of
soft quotas.

### 14. Unified cross-platform usage visualization (CR-30)

**Requirement**: Single view of token usage and chargeback across
on-prem clusters AND cloud providers (Vertex, Anthropic).

**Status**: We have Thanos/Grafana for cluster metrics. But
cross-provider cost views (combining internal GPU cost with
external API billing) need cost attribution data that spans
both Prometheus metrics and billing records.

### 15. Cost center mapping for chargeback (PT-57)

**Requirement**: Map user → cost center (from LDAP/HR data) for
internal cross-charging between departments.

**Status**: Not in design docs. Metering-service has the concept
(planned as nice-to-have). Requires integration with corporate
identity systems.

---

## Gaps in Pricetag infra (not in Grid scope but needed for production)

These are dogfood gaps that need resolution before production,
regardless of Grid migration:

| ID | Gap | Source |
|----|-----|--------|
| PT-66 | No Prometheus + Grafana alerting for gateway health | dogfood-cluster-layout |
| PT-80 | No database backup strategy | dogfood-cluster-layout |
| PT-81 | No monitoring/alerting pipeline | dogfood-cluster-layout |
| PT-82 | No network policies | dogfood-cluster-layout |
| PT-83 | No in-cluster TLS | dogfood-cluster-layout |
| PT-84 | No audit logging | dogfood-cluster-layout |
| PT-85 | No HPA (all single replica) | dogfood-cluster-layout |
| PT-89 | Stress test at 20K concurrent sessions not done | pricetag-plan |
| PT-90 | Latency benchmark vs direct provider not done | pricetag-plan |
| PT-91 | Feature parity (streaming, tool use, caching) not validated | pricetag-plan |
| PT-95 | Compliance review (SOC2, HIPAA, GDPR) not started | pricetag-mvp |
| PT-96 | DPIA assessment under GDPR Art. 35 not started | pricetag-mvp |
| PT-97 | Works council compliance for EU per-user tracking not started | pricetag-mvp |

---

## Requirements explicitly out of scope (confirmed)

These are in customer/Pricetag sources but correctly out of scope
for the Grid MVP:

| Requirement | Reason |
|-------------|--------|
| Batch processing (CR-20, CR-21, CR-42) | llm-d feature, not Grid/gateway |
| MIG support (CR-33) | GPU Operator / NFD, available today |
| Multi-LoRA serving (CR-34) | vLLM + llm-d EPP, not gateway |
| Semantic/cost-based routing (CR-45, CR-52) | Content classification is EPP-level, not gateway |
| Semantic cache (CR-53) | Separate concern |
| A/B testing of models (CR-57) | Virtual model endpoint, future |
| LoRA adapter routing (CR-31, CR-32) | llm-d EPP, not gateway |
| Prompt compression (PT-98) | Separate service (headroom) |

---

## Requirements we captured but may contradict customer needs

### Rate limiting scope mismatch

Our design says: "Rate limiting at MaaS level for MVP (Kuadrant/Limitador)"

Customer says: "Rate limits per application+model pair" with TPM, TPH,
concurrent caps, pre-charging, and refunds.

**The contradiction**: MaaS/Limitador does per-subscription per-model
limits with fixed windows. Customer wants per-APP (not per-subscription),
with refill semantics (not fixed reset), and concurrent request caps.
These are not the same thing. MaaS Limitador is a partial match at best.

### External metering vs token_rate_limit

Our migration plan assumes moving from `external_metering` to
`token_rate_limit`. But the customer's current system is more
sophisticated in some ways:
- Redis-cached enforcement (not per-request DB query)
- Input token calculation using model tokenizers at ingress
- Pre-charge based on input + max output (not just fixed estimate)

Praxis `token_rate_limit` with `input_plus_max_tokens` estimation
strategy addresses #3, but the Redis caching (#1) and tokenizer-based
estimation (#2) are not captured in our migration plan.

---

## Recommended Actions

### Must capture in design docs

1. **Concurrent request limits** — add as a gap with options (semaphore filter, connection limit)
2. **Hierarchical quotas** — add as post-MVP requirement, track Praxis #121
3. **Error refund behavior** — verify and document reserve/reconcile error path
4. **429 response headers** — verify token_rate_limit emits full set
5. **Quota warning thresholds** — add as requirement (80%/90% headers)
6. **Per-app vs per-user identity** — add as gap, depends on MaaS identity model
7. **Key rotation procedure** — add to gaps
8. **Service account keys** — add to gaps
9. **Token type preservation in migration** — add to rate-limiting-migration.md

### Should discuss

10. **GPU capacity-based quota calculation** — is this our concern or platform ops?
11. **Free tier fallback routing** — interesting use of soft quotas
12. **Cross-provider cost visualization** — Thanos handles metrics but not billing reconciliation
13. **Onboarding simulation tooling** — operational process, needs tooling
14. **Rate limiting scope mismatch** — MaaS Limitador vs customer's per-app model
