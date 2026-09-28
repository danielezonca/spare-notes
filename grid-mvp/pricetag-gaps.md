# Pricetag Dogfood — Production Gaps

Gaps specific to the Pricetag dogfood deployment. Separate from Grid
MVP gaps (see gaps.md). Where Grid deployment resolves a gap, it's
marked accordingly.

---

## Functional Gaps

| Gap | Description | Grid resolves? |
|-----|------------|----------------|
| **Per-user quota** | Single global `MONTHLY_TOKEN_QUOTA` for all users. Can't differentiate power users from light users | **Yes** — `token_rate_limit` with `key: authenticated_subject` provides per-user rules |
| **Cost reservation** | Coarse yes/no balance check, no per-request token reservation. Concurrent requests can overshoot budget | **Yes** — `token_rate_limit` reserve/reconcile prevents overshooting |
| **Quota warning thresholds** | No 80%/90% warning headers. Users get 429 with no advance notice | **Partially** — `token_rate_limit` emits `X-RateLimit-Remaining-Tokens` headers. Threshold-based warnings need additional work |
| **Quota exception endpoint** | No admin API to manually adjust a user's quota | **Yes** — per-user rules in `token_rate_limit` config, or MaaS `MaaSSubscription.tokenRateLimits` per CRD |
| **GLM free tier fallback** | 429 when frontier quota exhausted, no fallback to free on-prem model | **Partially** — soft enforcement (#1381) + intelligent routing could redirect. Needs design |
| **Service account keys** | No CI/CD key type with separate/no quota. Pipelines compete with human engineers | **No** — needs MaaS maas-api extension (new key type) |
| **SSO/OIDC for dashboard** | Dashboard uses session cookies, no Red Hat SSO | **Partially** — Grafana (via Thanos) replaces the dashboard and supports SSO natively. metering-service dashboard retired |

## Production-Readiness Gaps

| Gap | Description | Grid resolves? |
|-----|------------|----------------|
| **No DB backup** | API keys + metering data in unbackable StatefulSet PostgreSQL | **Yes** — shared managed DB (RDS or equivalent) with automated backups |
| **No monitoring/alerting** | Gateway outage not detected until user complaints | **Yes** — Thanos/Grafana via ACM Multi-cluster Observability |
| **No network policies** | Any pod can reach maas-api, DB | **No** — needs per-cluster NetworkPolicy configuration |
| **No in-cluster TLS** | Plaintext traffic between gateway ↔ maas-api ↔ DB | **No** — needs service mesh or per-service TLS config |
| **No audit logging** | No record of key creation, quota changes, admin actions | **No** — needs audit logging implementation |
| **No HPA** | Single replica on most components (maas-api, DB) | **Yes** — multi-cluster provides HA. Per-cluster HPA still recommended |
| **No stress test** | 20K concurrent sessions target not validated | **No** — needs performance testing |
| **No latency benchmark** | <100ms p95 overhead vs direct provider not validated | **No** — needs benchmarking |
| **Feature parity not validated** | Streaming, long context, tool use, caching assumed to work | **No** — needs integration testing |
| **No CI/CD pipeline** | Manual builds via `release.sh`, no git SHA pinning | **Partially** — ACM/GitOps pipeline covers CRD distribution. Build pipeline still needed |
| **Repo bus-factor** | metering-service in personal GitHub, not org repo | **No** — repo transfer needed |

## Compliance Gaps

| Gap | Description | Grid resolves? |
|-----|------------|----------------|
| **SOC2/HIPAA/GDPR review** | Not started. Required before production for 8K engineers | **No** — platform-level, independent of Grid |
| **DPIA assessment (GDPR Art. 35)** | Systematic monitoring of employees at scale requires DPIA | **No** — legal/compliance process |
| **Works council compliance** | EU per-user tracking requires works council approval | **No** — legal/HR process |

---

## Summary

| Category | Total gaps | Grid resolves | Remains |
|----------|-----------|--------------|---------|
| Functional | 7 | 4 fully, 2 partially | 1 (service account keys) |
| Production-readiness | 11 | 3 fully, 1 partially | 7 |
| Compliance | 3 | 0 | 3 |
| **Total** | **21** | **7 fully, 3 partially** | **11** |

Grid deployment resolves the most critical gaps: per-user quotas,
budget overshooting, DB backups, and monitoring. The remaining gaps
are infrastructure hardening (network policies, TLS, audit logging),
testing (stress test, latency benchmark), and compliance (GDPR, works
council) — all independent of Grid.
