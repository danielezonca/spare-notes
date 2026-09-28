# AI Grid — Personas

Three personas interact with the platform. Each has a distinct scope
of control, different auth requirements, and a different relationship
to the model catalog.

---

## Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Platform Admin                                 │
│  Owns infrastructure. Deploys models. Onboards tenants.            │
│  Sets default model catalog and per-tenant budgets.                │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Tenant Admin                              │   │
│  │  Operates within a tenant. Controls which models are         │   │
│  │  visible to their users. Manages user access and limits.     │   │
│  │                                                              │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │                 Tenant User                           │   │   │
│  │  │  Consumes the service. Gets keys, uses models,        │   │   │
│  │  │  views own usage. No admin capabilities.              │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. Platform Admin

**Who**: Infrastructure / operations team that owns and operates the
AI platform.

**Analogy**: Google Cloud team managing Vertex AI — they decide which
models exist on the platform, manage GPU capacity, and onboard
customers.

**Does NOT use the platform as an end user.** This persona builds and
maintains the service that the other two personas consume.

### Responsibilities

| Area | Actions |
|------|---------|
| **Infrastructure** | Deploy and manage clusters, GPUs, networking. Install Grid Operators, SWIM mesh, shared DB. Manage platform health and capacity |
| **Model deployment** | Deploy self-hosted models (vLLM instances). Register models in the platform catalog. Decide which models are available by default to all tenants |
| **Tenant onboarding** | Create tenant configuration when a new customer is registered. Assign initial model access and budget. Set per-tenant quotas |
| **Default catalog** | Curate the default model list — every tenant gets this set unless overridden. Can also grant special-case access (e.g., early access to a new model for a specific tenant) |
| **Platform fallbacks** | Optionally configure platform-level external provider fallbacks (scale to cloud when on-prem capacity is insufficient) |

### Auth

Cluster-level access: K8s RBAC, ACM/GitOps pipelines, direct CRD
management.

### What they manage (MaaS CRDs)

| CRD | Action |
|-----|--------|
| `Config` | Create cluster singleton — platform anchor |
| `AITenant` | Create per tenant — bootstraps namespace, Gateway ref, OIDC |
| `MaasTenantConfig` | Set tenant defaults: API key policy, telemetry, scaling |
| `LLMInferenceService` | Deploy self-hosted model backends (vLLM) |
| `MaaSModelRef` | Register a model in the catalog (references backend) |
| `ExternalModel` | Configure platform-level external providers (fallback) |
| `MaaSSubscription` | Set initial per-tenant budgets and model rate limits |

### Tooling

- Admin UI (RHCE) → Management API → ACM → clusters
- kubectl / GitOps for CRD management
- Grafana dashboards for platform-wide monitoring

---

## 2. Tenant Admin

**Who**: Team lead or designated admin within a tenant organization.

**Analogy**: Red Hat admin managing AI access for their business unit —
they don't deploy models, but they control who on their team can use
AI and at what access tier.

**No K8s access. No platform UI (MVP).** The Tenant Admin's primary
tool is the corporate identity provider (SSO/IdP).

### MVP vs Post-MVP

The Tenant Admin role evolves across phases:

#### MVP — SSO group management only

The Tenant Admin manages **who can access what** by adding and removing
users from SSO groups in the corporate IdP (Keycloak, LDAP, Red Hat
SSO). No platform-specific tooling required.

| Area | How |
|------|-----|
| **User access** | Add/remove users from SSO groups in the corporate IdP. Groups are mapped to model access and quotas by Platform Admin during onboarding |
| **Access tiers** | Move users between groups to change their access tier (e.g., `ai-team-a-standard` → `ai-team-a-premium` for Fable access) |
| **Cut off a user** | Remove them from all AI SSO groups |
| **Usage monitoring** | Grafana dashboards (tenant-scoped) |
| **Model visibility changes** | Request to Platform Admin (manual process) |
| **Quota changes** | Request to Platform Admin (manual process) |

**What the Platform Admin sets up during onboarding:**

```
SSO Group                    → MaaSAuthPolicy    → MaaSSubscription
─────────────────────────────────────────────────────────────────────
ai-team-a-standard           → Qwen3, Sonnet,    → 500K tokens/user/month
                               Llama, GPT-4o
ai-team-a-premium            → all standard +    → 2M tokens/user/month
                               Fable, Opus
ai-team-a-ci                 → Qwen3, Sonnet     → 5M tokens/month (no per-user)
                               (service account)    
```

The Tenant Admin doesn't touch CRDs — they manage group membership,
and the platform maps groups to access and quotas.

#### Post-MVP — thin self-service Admin UI

When tenant self-service becomes a requirement, add a lightweight
Admin UI for operations that SSO groups can't express:

| Area | Actions |
|------|---------|
| **Model visibility** | Disable/enable models from the platform catalog for this tenant (e.g., disable Fable to control costs) |
| **Quota tuning** | Redistribute the tenant's budget across groups. Adjust per-group rate limits within the ceiling Platform Admin set |
| **BYOK** | Add external models with tenant-owned API keys. Gateway does routing only — tenant owns the provider relationship and cost |
| **Bulk key revocation** | Revoke all API keys for a group or user via `POST /v1/api-keys/bulk-revoke` |

### Auth

- **MVP**: Corporate SSO/IdP admin access (to manage groups). No
  platform credentials needed
- **Post-MVP**: Corporate SSO for Admin UI login. No K8s access, no
  OpenShift account

### What they CANNOT do (both phases)

- Deploy or undeploy models
- Create new tenants
- Access the K8s API, CRDs, or cluster infrastructure
- See other tenants' data, users, or usage
- Exceed the budget ceiling set by Platform Admin

### Tooling

- **MVP**: Corporate IdP admin tools (Keycloak, LDAP, Red Hat SSO) +
  Grafana dashboards (tenant-scoped)
- **Post-MVP**: Add thin Admin UI for model visibility, quota tuning,
  BYOK

---

## 3. Tenant User

**Who**: Engineer using AI tools in their daily work.

**Analogy**: A Red Hat engineer using Claude Code, Codex CLI, or
custom scripts via the AI gateway. They consume the service — they
don't configure it.

### Responsibilities

| Area | Actions |
|------|---------|
| **API keys** | Create personal API keys (bound to subscription). List and revoke own keys |
| **Model discovery** | List available models (`GET /v1/models`) — sees only what Tenant Admin has enabled |
| **Model usage** | Send inference requests via `POST /v1/chat/completions` through the gateway |
| **Usage visibility** | View own token usage and remaining budget |

### Auth

- **For the portal (management APIs)**: SSO/OIDC (corporate IdP) or
  API key. No OpenShift account required
- **For inference (gateway)**: API key in `Authorization: Bearer sk-...`
  header

### What they interact with

| Endpoint | Purpose |
|----------|---------|
| `GET /v1/models` | Discover available models |
| `GET /v1/subscriptions` | View subscriptions and limits |
| `POST /v1/api-keys` | Create a new API key |
| `POST /v1/api-keys/search` | List own keys |
| `DELETE /v1/api-keys/:id` | Revoke own key |
| `POST /v1/chat/completions` | Use a model (via gateway) |
| Grafana dashboards | View personal usage |

### What they CANNOT do

- See or manage other users' keys or usage
- Enable / disable models
- Change rate limits or quotas
- Access any admin or infrastructure tooling
- Access the cluster directly (no kubectl, no CRDs)

---

## Persona Boundaries

### Model access flow (MVP)

```
Platform Admin                    Tenant Admin                  Tenant User
      │                                │                             │
      │  deploys models                │                             │
      │  maps SSO groups to models:    │                             │
      │  ai-team-a-standard → 4 models │                             │
      │  ai-team-a-premium  → 6 models │                             │
      │                                │                             │
      │                                │  manages SSO groups         │
      │                                │  in corporate IdP           │
      │                                │                             │
      │                                │  adds alice to              │
      │                                │  ai-team-a-standard         │
      │                                │                             │
      │                                ├───────────────────────────► │
      │                                │                             │  alice authenticates
      │                                │                             │  SSO resolves groups
      │                                │                             │  → sees 4 models
      │                                │                             │
      │                                │  moves alice to             │
      │                                │  ai-team-a-premium          │
      │                                │                             │
      │                                ├───────────────────────────► │
      │                                │                             │  → now sees 6 models
      │                                │                             │    (including Fable)
```

### Model access flow (Post-MVP)

```
Platform Admin                    Tenant Admin                  Tenant User
      │                                │                             │
      │  deploys 8 models globally     │                             │
      │  assigns all 8 to tenant       │                             │
      │                                │                             │
      ├──────────────────────────────► │                             │
      │                                │                             │
      │                                │  opens Admin UI             │
      │                                │  disables Fable, Opus       │
      │                                │  (cost control)             │
      │                                │                             │
      │                                │  SSO groups still control   │
      │                                │  WHO gets access            │
      │                                │                             │
      │                                ├───────────────────────────► │
      │                                │                             │  GET /v1/models
      │                                │                             │  → sees 6 models
```

### Budget flow

```
Platform Admin                    Tenant Admin                  Tenant User
      │                                │                             │
      │  sets budget per SSO group:    │                             │
      │  standard → 500K/user/month    │                             │
      │  premium  → 2M/user/month     │                             │
      │  tenant ceiling: 10M/month     │                             │
      │                                │                             │
      │                                │  (MVP) moves user between   │
      │                                │  groups to change tier      │
      │                                │                             │
      │                                │  (post-MVP) also tunes      │
      │                                │  per-group budgets via UI   │
      │                                │                             │
      │                                ├───────────────────────────► │
      │                                │                             │  uses 12K tokens
      │                                │                             │
      │                                │                             │  remaining:
      │                                │                             │  488K personal
      │                                │                             │  9.5M tenant
```

### Auth requirements

| Persona | K8s access | OpenShift account | Corporate SSO | API key |
|---------|-----------|-------------------|---------------|---------|
| Platform Admin | Full | Yes | Yes | No (uses kubectl/GitOps) |
| Tenant Admin | **None** | **No** | Yes (IdP admin for groups) | No |
| Tenant User | **None** | **No** | Yes (portal) | Yes (inference) |

---

## Cluster Visibility Assumptions

A core design constraint: **only Platform Admin has cluster access.**
Neither Tenant Admin nor Tenant User touches K8s. Everything
downstream follows from this.

### What each persona can see and touch

| Layer | Platform Admin | Tenant Admin | Tenant User |
|-------|---------------|-------------|-------------|
| **K8s API server** | Full access (cluster-admin or equivalent) | **No access** | No access |
| **CRDs** | Read + write all MaaS and Grid CRDs | **No access** — CRDs managed by Platform Admin | No access |
| **Grid CRDs** (`InferenceProvider`, `GridSite`, etc.) | Full read + write | No access | No access |
| **Secrets** (provider API keys, TLS certs) | Full access | No access (post-MVP: BYOK via Admin UI, not kubectl) | No access |
| **Pods / Deployments** | Full visibility | No access | No access |
| **Namespaces** | Create and manage all | No access | No access |
| **Node / GPU resources** | Full visibility — capacity planning, GPU allocation | No access | No access |
| **Helm / GitOps** | Manages releases, ACM policies, ArgoCD apps | No access | No access |
| **Corporate IdP (SSO groups)** | Creates group → model/quota mappings | **Admin access** — adds/removes users from groups | No access (member only) |

### What each persona sees through the HTTP layer

| Endpoint / UI | Platform Admin | Tenant Admin | Tenant User |
|---------------|---------------|-------------|-------------|
| **Admin UI (RHCE)** | Full — all tenants, all models, all clusters | MVP: no access. Post-MVP: tenant-scoped view | No access |
| **Management API** | Full — CRUD on all tenants | MVP: no access. Post-MVP: tenant-scoped | No access |
| **Corporate IdP admin** | Creates SSO group structure | **Primary tool** — manages group membership | No access (authenticates via SSO) |
| **maas-api** (`/v1/*`) | Not a primary consumer | Not a primary consumer | Self-service only: models, keys, subscriptions, usage |
| **Gateway** (`/v1/chat/completions`) | Not a primary consumer | Not a primary consumer | Primary consumer |
| **Grafana dashboards** | Platform-wide: all tenants, all clusters, capacity | Tenant-scoped: team usage, cost, rate limit utilization | Personal: own token usage and remaining budget |

### Hard boundaries (invariants)

1. **Neither Tenant Admin nor Tenant User touches K8s.** No kubectl,
   no kubeconfig, no CRD access, no namespace visibility. If a
   workflow requires either persona to run kubectl, the design is
   wrong.

2. **Tenant Admin never sees infrastructure.** They don't know how
   many clusters exist, how many GPUs are allocated, or which nodes
   run which models. Models are a catalog, not pods on nodes.

3. **Tenant Admin cannot escalate.** In MVP, the Tenant Admin has no
   platform access — only IdP admin access. Post-MVP, the Admin UI
   enforces tenant-scoped access at the application layer.

4. **Platform Admin doesn't use API keys.** Platform Admin interacts
   via kubectl, GitOps, ACM, and the Admin UI — never via the gateway
   with an API key. If Platform Admin needs to test inference, they
   create a test tenant and use it as a Tenant User.

5. **Cross-tenant isolation.** Tenant Admin A cannot see Tenant B's
   users, keys, usage, or subscriptions. SSO groups are per-tenant.
   The shared DB stores all tenants' data, but maas-api enforces
   tenant-scoped queries.

---

## Mapping to Current Implementation

The current MaaS design conflates Platform Admin and Tenant Admin into
a single "Tenant Admin" persona. This document separates them:

| Current design-notes.md | This document |
|--------------------------|---------------|
| "Tenant Admin" deploys models + manages access | Split: Platform Admin deploys, Tenant Admin manages SSO groups |
| Two personas | Three personas |
| Tenant Admin does `kubectl apply MaaSModelRef` | Platform Admin does this |
| Tenant Admin does `kubectl apply MaaSSubscription` | Platform Admin does this + maps to SSO groups |
| Tenant Admin does `kubectl apply MaaSAuthPolicy` | Platform Admin does this + maps to SSO groups |
| Tenant Admin uses Admin UI | MVP: no Admin UI for Tenant Admin. Post-MVP: thin self-service UI |

### SSO group → CRD mapping (managed by Platform Admin)

The Platform Admin creates the linkage between corporate identity and
platform access during tenant onboarding:

```
Corporate IdP                    Platform (K8s CRDs)
──────────────────────────────────────────────────────
SSO Group: ai-team-a-standard  → MaaSAuthPolicy
                                   subjects: [ai-team-a-standard]
                                   modelRefs: [qwen3, sonnet, llama, gpt4o]
                                 MaaSSubscription
                                   ownerGroups: [ai-team-a-standard]
                                   tokenRateLimits: 500K/user/month

SSO Group: ai-team-a-premium   → MaaSAuthPolicy
                                   subjects: [ai-team-a-premium]
                                   modelRefs: [qwen3, sonnet, llama, gpt4o, fable, opus]
                                 MaaSSubscription
                                   ownerGroups: [ai-team-a-premium]
                                   tokenRateLimits: 2M/user/month
```

The Tenant Admin manages group membership. The platform enforces
access and quotas based on that membership.

### External providers — two modes

External model configuration depends on intent:

| Mode | Persona | Mechanism | Phase | Gateway role |
|------|---------|-----------|-------|-------------|
| **Platform fallback** | Platform Admin | `ExternalModel` CR with platform-owned credentials. Configured for capacity overflow (scale to cloud when GPUs are saturated) | MVP | Full pipeline: routing + rate limiting + metering |
| **BYOK (bring your own key)** | Tenant Admin | `ExternalModel` CR with tenant-owned credentials via Admin UI. Tenant owns the provider relationship and cost | Post-MVP | Routing only — no platform rate limiting or metering on these requests |
