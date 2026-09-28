# Tokenomics Research — Praxis AI Strategy

Analysis of the Praxis AI team's current strategy around token accounting,
rate limiting, metering, and cost attribution. Based on Epic #656 and
related issues in [praxis-proxy/ai](https://github.com/praxis-proxy/ai).

## Sources

| Issue | Title | State | Milestone | Link |
|-------|-------|-------|-----------|------|
| #656 | Epic: Tokenomics | Open | v1.0.0 | [View](https://github.com/praxis-proxy/ai/issues/656) |
| #121 | Epic: Token Rate Limiting | Open | — | [View](https://github.com/praxis-proxy/ai/issues/121) |
| #78 | Epic: AI Cost and Budget Controls | Open | — | [View](https://github.com/praxis-proxy/ai/issues/78) |
| #577 | feat: external metering filter | Open | v0.6.0 | [View](https://github.com/praxis-proxy/ai/issues/577) |
| #707 | feat: API key validation filter | Open | v0.5.0 | [View](https://github.com/praxis-proxy/ai/issues/707) |
| #979 | Simultaneous per-user + per-model limits | Open | — | [View](https://github.com/praxis-proxy/ai/issues/979) |
| #1122 | Pre-forward token ceiling | Open | v0.5.0 | [View](https://github.com/praxis-proxy/ai/issues/1122) |
| #1123 | First-party cost/usage dashboards | Open | v0.6.0 | [View](https://github.com/praxis-proxy/ai/issues/1123) |
| #1201 | Reasoning tokens in CloudEvents | Open | — | [View](https://github.com/praxis-proxy/ai/issues/1201) |
| #1283 | Entitlement claim fencing (match_claims) | Open | — | [View](https://github.com/praxis-proxy/ai/issues/1283) |
| #1332 | Pre-forward token ceiling guard | Open | v0.5.0 | [View](https://github.com/praxis-proxy/ai/issues/1332) |
| #1334 | Compositional M5 bucket keys | Open | v0.5.0 | [View](https://github.com/praxis-proxy/ai/issues/1334) |
| #1359 | WebSocket upgrade bypass | Open | — | [View](https://github.com/praxis-proxy/ai/issues/1359) |
| #1378 | Replace Valkey Lua with plain commands | Open | v0.4.0 | [View](https://github.com/praxis-proxy/ai/issues/1378) |
| #1381 | Soft/shadow over-quota enforcement | Open | — | [View](https://github.com/praxis-proxy/ai/issues/1381) |
| #796 | Sliding window + token bucket (M1/M2/M6) | Merged | — | [View](https://github.com/praxis-proxy/ai/issues/796) |
| #790 | Shared Valkey quota enforcement | Closed | — | [View](https://github.com/praxis-proxy/ai/issues/790) |
| #1008 | Configurable estimation strategies | Merged | — | [View](https://github.com/praxis-proxy/ai/issues/1008) |
| #1132 | Token-type weights at reconciliation | Merged | — | [View](https://github.com/praxis-proxy/ai/issues/1132) |

## Documents

| Document | Description |
|----------|------------|
| [praxis-tokenomics-strategy.md](praxis-tokenomics-strategy.md) | Full analysis of Praxis AI's tokenomics strategy, feature roadmap, and implications for our migration |
| [praxis-demos-analysis.md](praxis-demos-analysis.md) | Analysis of 4 experimental demos from [praxis-proxy/experimental](https://github.com/praxis-proxy/experimental). Config shape evolution (Gen 1→4), feature matrix, practical guidance |


| [adversarial-review.md](adversarial-review.md) | Cross-reference gap analysis: 59 customer requirements + 98 Pricetag requirements vs design docs + implementations. **38 uncaptured gaps found**, 7 critical, 8 important |

Additional customer-sourced research (meeting notes, requirements,
RFEs) is in `other_sources/` — gitignored, local only.
