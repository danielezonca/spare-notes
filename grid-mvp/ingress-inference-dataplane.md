# AI Grid — Ingress & Inference Data Plane Architecture

Gap analysis, data plane mapping, adversarial evaluation, and
component breakdown for the Region Ingress + Inference topology.

Ingress clusters run MaaS Gateway + maas-api in addition to the shared
Grid stack. Inference clusters run Grid GW + standalone GW + llm-d for
inference only. The routing and model deployment model remain
symmetric — any cluster can host models.

---

## 1. Goal Matrix

Every architectural requirement mapped to the target design.

| ID | Requirement | Design Response |
|----|-------------|-----------------|
| R1 | **Region Ingress clusters** as user-facing entry points | Ingress cluster runs: MaaS GW + maas-api + Grid GW + Grid Operator + (optionally) standalone GW + llm-d. Customer traffic enters here. |
| R2 | **Inference clusters** for workloads | Inference cluster runs: standalone GW + llm-d + Grid GW + Grid Operator. No MaaS GW, no maas-api. May be air-gapped from public internet. |
| R3 | **A single cluster can be both ingress and inference** | No hard requirement to separate them. An ingress cluster is a superset — it can also host llm-d models with a standalone GW. When both roles are on one cluster, it runs the full stack. Ingress/inference is about MaaS presence, not model placement. |
| R4 | **Grid Proxy** for cross-cluster routing | AI Grid GW (Praxis AI) with `intelligent_route` + `load_balancer`. mTLS between Grid GWs at different sites. Grid GW does routing only — inter-service auth via SA tokens, not user API keys. |
| R5 | **`maas-api` stability** | Prefer existing maas-api APIs. Minor code changes acceptable to simplify integration (e.g., adding a field to the validation response). Integration via existing REST APIs (`/internal/v1/api-keys/validate`) and existing K8s informer (MaaSModelRef CRs created by reconciler). |
| R6 | **MaaS First** — all traffic through MaaS | Customer → MaaS GW → Grid GW → target. MaaS handles auth, rate limiting, metering before Grid routes. |
| R7 | **Unified ExternalModel abstraction** | All models (llm-d internal + third-party) registered as `ExternalModel` CRs in MaaS. One CR per distinct model name. |
| R8 | **Opt-in llm-d discovery** | llm-d deployments must carry annotation (e.g., `grid.praxis-proxy.io/managed: "true"`) AND live in a namespace matching Grid's filter. Unannotated deployments are invisible. |
| R9 | **Subscription replication across ingress clusters** | MaaS CRDs replicated via ACM/GitOps. Shared PostgreSQL DB for API key hashes across ingress clusters. |
| R10 | **llm-d → InferenceProvider reconciliation** | `ai grid` reconciler watches opted-in llm-d deployments, creates `InferenceProvider` CRs (`grid.praxis-proxy.io/v1alpha1`). |
| R11 | **State synchronization across clusters** | SWIM gossip (membership, capabilities, provider state) + signals polling (load metrics). |
| R12 | **External model reconciler** in Grid | Distinct reconciler inside Grid operator creates/updates `ExternalModel` + `MaaSModelRef` CRs on ingress clusters from InferenceProvider state. One `ExternalModel` per distinct model name. Runs on ingress clusters only. |

---

## 2. Data Plane Mapping

### 2.1 Request Flow

```
Customer → MaaS GW (auth, rate limit, metering, cred inject) → Grid GW (route only) → target GW → backend
```

Key properties of this flow:
1. Grid GW does routing only — no `api_key_auth`, no identity
   resolution. MaaS handles all authentication.
2. Rate limiting happens before routing — a rate-limited request never
   reaches Grid, reducing unnecessary cross-cluster traffic.
3. MaaS credential injection targets either external APIs directly
   or the Grid GW endpoint (for llm-d models via `ExternalModel`).

### 2.2 Detailed Request Flows

#### Flow A — llm-d model on the same ingress cluster

```
Customer
  → MaaS GW (Ingress A)
      ├── Authorino: validate API key via maas-api, resolve subscription
      ├── Limitador: token rate limit (per-user, per-model)
      ├── credential injection: ExternalModel credentialRef → Grid service token
      └── forward to Grid GW (Ingress A) endpoint
           └── model_to_header: extract model from body
           └── intelligent_route: select local candidate (same site)
           └── load_balancer: forward to standalone GW (Ingress A)
                └── EPP → vLLM
```

#### Flow B — llm-d model on a inference cluster

```
Customer
  → MaaS GW (Ingress A)
      ├── [auth, rate limit, metering — same as Flow A]
      └── forward to Grid GW (Ingress A)
           └── intelligent_route: select remote candidate (Inference cluster X)
           └── load_balancer → [mTLS] → Grid GW (Inference cluster X)
                └── forward to standalone GW (Inference cluster X)
                     └── EPP → vLLM
```

#### Flow C — llm-d model on a different ingress cluster

```
Customer
  → MaaS GW (Ingress A)
      ├── [auth, rate limit, metering — same as Flow A]
      └── forward to Grid GW (Ingress A)
           └── intelligent_route: select remote candidate (Ingress B)
           └── load_balancer → [mTLS] → Grid GW (Ingress B)
                └── forward to standalone GW (Ingress B)
                     └── EPP → vLLM
```

#### Flow D — Third-party external model (e.g., OpenAI)

```
Customer
  → MaaS GW (Ingress A)
      ├── [auth, rate limit, metering]
      ├── credential injection: ExternalModel credentialRef → OpenAI API key
      └── forward directly to api.openai.com
           (no Grid involvement — fixed external endpoint)
```

Third-party models bypass Grid routing because their endpoint is a
fixed public URL with no cross-cluster decision. They are still
registered as `ExternalModel` CRs for the unified model abstraction
(subscriptions, model listing, governance) but their traffic path
does not traverse the Grid GW.

#### Flow E — Auth failure

```
Customer
  → MaaS GW (Ingress A)
      ├── Authorino: API key invalid or subscription missing
      └── HTTP 401/403 (never reaches Grid GW)
```

#### Flow F — Rate limit exhausted

```
Customer
  → MaaS GW (Ingress A)
      ├── Authorino: valid
      ├── Limitador: budget exhausted
      └── HTTP 429 (never reaches Grid GW)
```

### 2.3 Filter Chain Changes

#### Grid GW — Target (routing only)

```yaml
filter_chains:
  - name: grid-route
    filters:
      - filter: model_to_header      # body → X-Gateway-Model-Name
      - filter: intelligent_route    # overlay candidates, site selection
        local_site: <this-site>
      - filter: load_balancer        # → selected target GW
```

Removed from Grid GW: `api_key_auth` (MaaS handles auth).
Removed from Grid GW: `token_rate_limit` (MaaS handles limits).

The Grid GW trusts requests from MaaS GW based on network-level
controls (same-cluster service routing, or mTLS for cross-cluster).

#### MaaS GW — Target

The MaaS GW pipeline is unchanged in structure — Authorino for auth,
Limitador for rate limits, credential injection from ExternalModel.

The change is what the `ExternalModel` endpoint resolves to:
- **llm-d models**: endpoint = Grid GW Kubernetes service
  (e.g., `grid-gw.grid-system.svc.cluster.local`). This resolves
  locally on each ingress cluster to the co-located Grid GW.
- **Third-party APIs**: endpoint = external API FQDN (e.g.,
  `api.openai.com`). Existing behavior, unchanged.

### 2.4 Cluster Stack Comparison

| Component | Ingress cluster | Inference cluster |
|-----------|------------|---------------|
| MaaS GW (Praxis AI) | Yes — customer entry point | No |
| maas-api | Yes — key validation, model listing | No |
| maas-controller | Yes — CRD reconciliation | No (CRDs replicated via ACM) |
| Grid GW (Praxis AI) | Yes — routing | Yes — receives cross-cluster traffic |
| Grid Operator | Yes — SWIM, signals, overlay | Yes — SWIM, signals, overlay |
| Standalone GW | Optional — if hosting models | Yes — serves llm-d models |
| llm-d (EPP + vLLM) | Optional — if hosting models | Yes |
| `ai grid` reconciler | Yes (if hosting models) | Yes |
| `external model` reconciler | Yes | No (no MaaS CRDs to create) |
| Shared DB (PostgreSQL) | Connected | Not connected (no maas-api) |

### 2.5 Model Registration Chain

The deployment and reconciliation workflow, step by step:

```
Step 1: Platform admin deploys llm-d
  └── kubectl apply: vLLM deployment, EPP, standalone Gateway
  └── Adds annotation: grid.praxis-proxy.io/managed: "true"
  └── Deploys in namespace matching Grid's filter (e.g., ai-models)
  └── Can happen on ANY cluster (ingress or inference)

Step 2: ai grid reconciler (same cluster)
  └── Watches: Gateway CRDs with opt-in annotation in filtered namespaces
  └── Creates: InferenceProvider CR
        gridNetworkRef: <network>
        backendKind: "local"
        endpoint: <standalone-gw-address>
        models: [{name: "facebook/opt-125m"}]
        metricsConfig: {signalNames: {queueDepth: "llm_d_epp_average_queue_size"}}

Step 3: SWIM state sync
  └── Grid Operator gossips ProviderState across all sites
  └── Every site's overlay now includes the new model's candidates
  └── Grid GWs hot-reload — can route to the new model

Step 4: external model reconciler (ingress clusters only)
  └── Watches: InferenceProvider CRs (reflects full mesh via SWIM)
  └── Deduplicates: groups by model name across all providers
  └── Creates (if not exists): ExternalModel CR
        provider: "grid"  (or "llm-d")
        targetModel: "facebook/opt-125m"
        endpoint: grid-gw.grid-system.svc.cluster.local
        credentialRef: {name: "grid-service-token"}
  └── Creates (if not exists): MaaSModelRef CR
        modelRef: {kind: ExternalModel, name: "facebook-opt-125m"}
  └── One ExternalModel + one MaaSModelRef per distinct model name
  └── MaaSSubscription + MaaSAuthPolicy are separate admin concerns
      (managed via ACM/GitOps, not auto-created)
```

---

## 3. Adversarial & Failure Analysis

### 3.1 Network Partition and Air-Gapped Inference Cluster Behavior

**Premise**: Inference clusters may be air-gapped from the public internet but
MUST maintain private network connectivity to other clusters (ingress
and inference). SWIM gossip and signals polling operate over this
private mesh.

Sources: Grid SWIM implementation
(`praxis-proxy/grid/swim/src/runtime.rs`), mTLS between Grid GWs
(`praxis-proxy/grid/certs/`).

| Scenario | Impact | Mitigation |
|----------|--------|------------|
| **Ingress → inference cluster private link drops** | Grid GW on ingress cluster cannot route to that inference cluster models. Overlay marks candidates as stale (`fresh=false`). If `staleCandidateTtlSeconds` is configured, candidates are eventually evicted. Requests for models only on that inference cluster fail 503. | SWIM detects partition via failed probes (`suspicion_timeout: 10s` default). Inference cluster models on other sites still reachable. Multi-site model deployment provides redundancy. |
| **Inference cluster loses public internet** | No impact on Grid. SWIM, signals, and data-plane traffic use private mesh. Inference cluster remains fully functional. | By design — inference clusters are expected to be air-gapped from public internet. |
| **Ingress-to-ingress cluster link drops** | Each ingress cluster routes only to inference clusters it can reach. Users on ingress cluster A may see different model availability than ingress cluster B. Shared DB still accessible (separate network path to RDS). | SWIM gossip partitions gracefully — each partition converges independently. When the link recovers, CRDTs merge deterministically (commutative, associative, idempotent). Source: `praxis-proxy/grid/crdt/src/grid_state.rs`. |
| **Shared DB unreachable from an ingress cluster** | maas-api on that ingress cluster cannot validate API keys. All requests fail 500/503 at MaaS GW. Grid routing still works but no traffic reaches it. | DB HA via RDS multi-AZ. Ingress cluster health checks should include DB connectivity. Load balancer removes unhealthy ingress cluster from DNS. |

**Backoff and recovery**:
- SWIM uses configurable probe interval (default `5s`) and suspicion
  timeout (default `10s`). Dead members are evicted after the
  suspicion window. Source: `GridNetworkSpec.swim` in
  `praxis-proxy/grid/operator/src/crd/grid_network.rs`.
- Grid Operator's `stale_candidate_ttl_seconds` field controls when
  stale remote candidates are removed from the overlay (default:
  retained indefinitely; recommended production value: `3600`).
- No store-and-forward or queueing — partitioned traffic fails fast
  with 503, which is correct behavior for a real-time inference
  gateway.

### 3.2 Subscription Sync Race Conditions Across Multi-Ingress-Cluster

**Consistency model**:
- API keys: immediately consistent (shared PostgreSQL DB across ingress clusters).
- MaaS CRDs (`MaaSSubscription`, `MaaSAuthPolicy`, `ExternalModel`,
  `MaaSModelRef`): eventually consistent via ACM/GitOps replication.
  Propagation delay: seconds to low minutes.
- Grid state (`InferenceProvider`, overlay): eventually consistent
  via SWIM gossip. Propagation delay: sub-second to seconds.

| Race Condition | Window | Impact | Severity |
|----------------|--------|--------|----------|
| **New subscription created on ingress cluster A, user hits Ingress B before GitOps sync** | Seconds to minutes | User sees models from old subscription. New models not yet available on ingress cluster B. | Low. Subscription changes are infrequent admin operations. User retries after sync. |
| **Model deployed on Inference cluster X, external model reconciler on ingress cluster A creates CRs before Ingress B gets them** | GitOps propagation time | Users on ingress cluster A see the new model immediately. Users on ingress cluster B see it after GitOps sync. | Low. Model deployments are admin operations, not user-facing latency. |
| **API key created on ingress cluster A, used on ingress cluster B** | Zero (shared DB) | Key validates immediately on any ingress cluster. | None. Shared DB ensures immediate consistency for keys. |
| **API key revoked on ingress cluster A, used on ingress cluster B** | Zero (shared DB) | Revocation is immediate across all hubs. | None. Critical safety property preserved. |
| **MaaSAuthPolicy updated on ingress cluster A, request on ingress cluster B** | GitOps propagation time | Ingress B enforces old auth policy until sync. User might access a model they should no longer have access to, or be denied a model they should now have access to. | Medium. Mitigate by ensuring security-critical changes (revocations) use the shared DB path, not CRD-only. |

**Mitigation pattern**: Security-critical operations (key revocation,
account suspension) must use the shared DB path, not CRD-only.
maas-api already does this — key validation hits the DB on every
request. Access expansion (granting new model access) is safe to be
eventually consistent because the worst case is a temporary denial,
not a leak.

### 3.3 Unannotated or Unauthorized llm-d Deployment Leaks

**Threat**: An llm-d deployment exists on a cluster but is not
managed by Grid. It consumes GPU resources but is unreachable
through the platform. Or conversely, an unauthorized deployment
gets picked up by Grid.

| Scenario | Gate | Outcome |
|----------|------|---------|
| llm-d deployed WITHOUT opt-in annotation | Annotation gate | Invisible to `ai grid` reconciler. No InferenceProvider created. Model does not appear in Grid overlay or MaaS model listing. Runs but is orphaned. |
| llm-d deployed WITH annotation but OUTSIDE filtered namespace | Namespace gate | Invisible to `ai grid` reconciler. Same outcome as above. |
| llm-d deployed WITH annotation AND in filtered namespace | Both gates pass | `ai grid` reconciler creates InferenceProvider. Model propagates via SWIM. `external model` reconciler creates ExternalModel on ingress clusters. Model visible to users with appropriate subscriptions. |
| Annotation removed from existing deployment | Annotation gate | `ai grid` reconciler detects removal, deletes InferenceProvider. SWIM propagates deletion. `external model` reconciler removes ExternalModel if no other provider serves this model. Model disappears from catalog. |
| Unauthorized deployment in filtered namespace with annotation | Both gates pass | Model enters the platform. **Risk**: anyone with namespace write access can inject a model. |

**Mitigation for unauthorized deployment**:
- Namespace filter should target dedicated, RBAC-controlled namespaces
  (not broad namespaces like `default`).
- The `ai grid` reconciler should validate the Gateway CRD against
  an allowlist or require a secondary approval (e.g., a Grid-managed
  label that only the Grid operator can set).
- Platform admin monitors `InferenceProvider` creation via K8s events
  or audit logs.

**Observability gap**: The `ai grid` reconciler should emit Kubernetes
events or metrics for:
- Unannotated llm-d deployments in filtered namespaces (warning:
  "deployment exists but not managed by Grid").
- InferenceProvider creation/deletion (info: audit trail).
- Annotation/namespace mismatches (debug: troubleshooting).

### 3.4 Proxy Overhead on Streaming LLM Token Delivery

**Hop count comparison**:

| Flow | Previous Design | Ingress & Inference |
|------|----------------|-------------|
| Local model (same cluster) | Customer → Grid GW → MaaS GW → EPP → vLLM (4 hops) | Customer → MaaS GW → Grid GW → standalone GW → EPP → vLLM (5 hops) |
| Remote model (cross-cluster) | Customer → Grid GW → [mTLS] → MaaS GW → EPP → vLLM (4 hops + mTLS) | Customer → MaaS GW → Grid GW → [mTLS] → Grid GW → standalone GW → EPP → vLLM (6 hops + mTLS) |
| External API | Customer → Grid GW → MaaS GW → external API (3 hops) | Customer → MaaS GW → external API (2 hops, Grid bypassed) |

Ingress & Inference adds 1 hop for local models, 2 hops for cross-cluster
models, and removes 1 hop for external APIs.

**Impact on streaming (SSE)**:
- Each proxy hop adds latency for the **initial connection setup** only
  (TCP + TLS handshake). For streaming, once the connection is
  established, tokens flow through the proxy chain with negligible
  per-token overhead — Praxis is an event-driven proxy designed for
  this. Source: `praxis-proxy/ai/server/src/server.rs`.
- The additional hop for local models is intra-cluster (sub-millisecond
  network latency). The impact on time-to-first-token is negligible.
- The additional hop for cross-cluster models is one more intra-cluster
  hop on the remote side. The dominant latency is the mTLS
  cross-cluster hop, which is unchanged.
- mTLS overhead between Grid GWs is already accounted for in the
  existing Grid design. Source:
  `praxis-proxy/grid/certs/src/generate.rs`.

**Mitigation**:
- Connection pooling at each proxy layer (Praxis supports this).
- HTTP/2 multiplexing reduces handshake overhead for concurrent
  requests to the same target.
- For external APIs, the hop count is actually reduced (Grid bypassed).

**Verdict**: The additional proxy hops are architecturally clean and
have minimal performance impact for streaming workloads. The
dominant latency in inference requests is model computation time,
not proxy hops.

---

## 4. Proposed Design Choices & Component Breakdown

### 4.1 New Components

#### 4.1.1 `ai grid` Reconciler

**Purpose**: Watches opted-in llm-d Gateway deployments and creates
`InferenceProvider` CRs in the Grid API group.

**Location**: Grid Operator (`praxis-proxy/grid/operator/`). New
controller module alongside existing `inference_provider`,
`grid_network`, `grid_site` controllers.

**Language**: Rust (matches Grid Operator).

**Watches**:
- Gateway CRDs (gateway-api `Gateway` resources) in configured
  namespaces.
- Filters by annotation: `grid.praxis-proxy.io/managed: "true"`.
- Namespace filter: configurable list or label selector in
  `GridNetwork` spec.

**Creates**:
- `InferenceProvider` CR per opted-in llm-d deployment.
- Maps: model names from Gateway/InferencePool metadata →
  `spec.models`.
- Maps: standalone GW endpoint → `spec.endpoint`.
- Sets: `backendKind: "local"`, `metricsConfig` from llm-d EPP
  endpoint.

**Lifecycle**:
- Create InferenceProvider when annotated Gateway appears.
- Update when Gateway spec changes (model list, endpoint, replicas).
- Delete when Gateway is removed or annotation is removed.
- Requeue on error with exponential backoff.

**Configuration**:
- Annotation key (default: `grid.praxis-proxy.io/managed`).
- Annotation value (default: `"true"`).
- Namespace filter (list or label selector).
- `GridNetwork` reference for created InferenceProviders.

#### 4.1.2 `external model` Reconciler

**Purpose**: Creates/updates `ExternalModel` and `MaaSModelRef` CRs
in the MaaS API group, driven by InferenceProvider state. Ensures
every distinct model served by Grid is visible in MaaS.

**Location**: Grid Operator (`praxis-proxy/grid/operator/`). New
controller module. Runs only on ingress clusters (guarded by a feature
flag or role label).

**Language**: Rust (matches Grid Operator). Uses `kube-rs` to create
CRs in the `maas.opendatahub.io/v1alpha1` API group.

**Watches**:
- `InferenceProvider` CRs (local + replicated via SWIM/CRDT state).
- The reconciler sees the full mesh model catalog because SWIM
  gossip propagates provider state to every site.

**Creates** (on ingress clusters only):
- `ExternalModel` CR — one per distinct model name across all
  InferenceProviders.
  - `spec.provider`: `"grid"` (or configurable).
  - `spec.targetModel`: the model name (e.g., `"facebook/opt-125m"`).
  - `spec.endpoint`: local Grid GW K8s service
    (`grid-gw.grid-system.svc.cluster.local`).
  - `spec.credentialRef`: reference to a Grid service token Secret.
- `MaaSModelRef` CR — one per ExternalModel.
  - `spec.modelRef.kind`: `"ExternalModel"`.
  - `spec.modelRef.name`: the ExternalModel CR name.

**Does NOT create**:
- `MaaSSubscription` — admin decision (who gets access, what limits).
- `MaaSAuthPolicy` — admin decision (authorization subjects).
- These are managed by Platform Admin via ACM/GitOps.

**Deduplication logic**:
- Model name is the deduplication key.
- If `facebook/opt-125m` exists on 3 clusters → 3 InferenceProviders →
  1 ExternalModel + 1 MaaSModelRef.
- The Grid overlay handles multi-site routing via multiple candidates
  for the same model.

**Lifecycle**:
- Create ExternalModel when a new model name appears in any
  InferenceProvider.
- No-op when additional InferenceProviders for the same model appear
  (deduplication).
- Delete ExternalModel when the last InferenceProvider serving that
  model is removed.
- Requeue on error with exponential backoff.

**Guard**: Must only run on ingress clusters. Options:
- Feature flag in Grid Operator configuration.
- Node/cluster label selector.
- Presence of MaaS CRDs on the cluster (fail gracefully if absent).

#### 4.1.3 Inter-Service Auth

**Purpose**: Authentication between gateway layers. Requests from
MaaS GW → Grid GW and Grid GW → standalone llm-d GW must be
authenticated with service-level credentials, not user API keys.
Cross-cluster Grid GW → Grid GW traffic is already authenticated
via Grid mTLS.

**Design options**:

| Option | How | Pros | Cons |
|--------|-----|------|------|
| **K8s ServiceAccount token** (recommended) | MaaS GW pod mounts a projected SA token. ExternalModel `credentialRef` points to it. Grid GW validates via TokenReview. Same pattern for Grid GW → llm-d GW. | Standard K8s auth. No manual secret management. Integrates with RBAC. Automatic rotation. | Requires TokenReview API access on receiving side. |
| **mTLS via service mesh** | Mutual TLS between services using mesh-issued certificates (e.g., Istio, SPIFFE). | Zero-touch auth. Certificate lifecycle managed by mesh. | Requires service mesh. Adds infrastructure dependency. |
| **Static shared secret** | Pre-shared token in a K8s Secret. Sender injects it. Receiver validates as bearer check. | Simple. No external dependencies. | Manual rotation. No fine-grained authorization. |

**Recommendation**: ServiceAccount tokens (option 1). Each gateway
validates incoming requests via K8s TokenReview. The SA token is
projected into the sender pod and referenced by the ExternalModel
`credentialRef`. This provides defense-in-depth without manual
credential management.

### 4.2 Grid GW Filter Chain

```yaml
filters:
  - filter: model_to_header      # body → X-Gateway-Model-Name
  - filter: intelligent_route    # overlay candidates
  - filter: load_balancer        # → selected target GW
```

Grid GW does routing only. MaaS GW handles all auth upstream.

**Repo**: `praxis-proxy/ai/` — configuration change only, no filter
code changes.

### 4.3 Grid Operator CRD: Namespace Filter

New field on `GridNetwork` spec (or a new CRD) to configure llm-d
discovery:
- `llmdDiscovery.namespaces`: list of namespaces to watch.
- `llmdDiscovery.annotation`: annotation key/value to match.

`InferenceProvider` uses `siteSelector.matchLabels` for site-level
filtering. The namespace filter is an additional gate for llm-d
discovery specifically.

**Repo**: `praxis-proxy/grid/operator/src/crd/grid_network.rs` (or
new CRD).

### 4.4 ExternalModel CRD Usage

The `ExternalModel` CRD schema does NOT need modification. The
`provider` field is a free-form string — the maas-controller handles
ExternalModel reconciliation generically.

For llm-d models:
- `provider`: `"grid"` (convention signaling Grid-routed).
- `targetModel`: model name (e.g., `"facebook/opt-125m"`).
- `endpoint`: Grid GW K8s service FQDN.
- `credentialRef`: Secret containing projected SA token for
  service-level auth.

For third-party APIs:
- `provider`: `"openai"`, `"anthropic"`, etc.
- `targetModel`: upstream model name.
- `endpoint`: external API FQDN.
- `credentialRef`: API key Secret.

### 4.5 Unaffected Components

| Component | Notes |
|-----------|-------|
| `maas-api` | Hard constraint. No code changes. Existing APIs sufficient. |
| SWIM gossip protocol | Propagates InferenceProvider state between all sites. |
| Signals polling | Load metrics between Grid Operators. |
| Scoring engine | Scores and ranks candidates in the overlay. |
| CRDT state propagation | Merges provider records across sites. |
| `GridSite` CRD and controller | Site discovery and mTLS establishment. |
| `GridNetwork` CRD and controller | Overlay generation. Namespace filter addition needed (Section 4.3). |
| `InferenceProvider` CRD | Schema supports all needed fields. |
| Kuadrant (Authorino + Limitador) | Auth and rate limiting at MaaS GW. |

---

## 5. Open Questions

Issues to resolve before Phase 2 implementation.

| ID | Question | Options | Recommendation |
|----|----------|---------|----------------|
| OQ-1 | **What K8s resource does the `ai grid` reconciler watch?** The task says "llm-d deploys Gateway." Is this a gateway-api `Gateway` resource, an `InferencePool`, an `InferenceModel`, or a custom llm-d CRD? | (a) `Gateway` resource (gateway-api). (b) `InferencePool` (gateway-api-inference-extension). (c) Custom llm-d CRD. | Depends on what llm-d actually creates. Need to confirm with llm-d team. |
| OQ-2 | **What Secret does the ExternalModel `credentialRef` reference?** The `external model` reconciler creates ExternalModel CRs with `provider: "grid"`. The `credentialRef` needs to point to a Secret containing the SA token for MaaS GW → Grid GW auth. | (a) A pre-created Secret per Grid network containing the projected SA token. (b) The reconciler references the MaaS GW pod's projected SA token volume. (c) A Secret auto-generated by the reconciler. | (a) — a pre-created Secret that the MaaS GW pod mounts. The reconciler references it by convention. |
| OQ-3 | **How does maas-controller handle `provider: "grid"` ExternalModels?** The existing controller creates HTTPRoutes and credential injection for external APIs. For Grid-routed models, it needs to create an HTTPRoute pointing to the Grid GW service. | (a) maas-controller handles it generically (endpoint is just a URL). (b) maas-controller needs a Grid-aware code path. | (a) — the controller should already work with any endpoint FQDN. Verify. |
| OQ-4 | **Should the `external model` reconciler also handle third-party external models?** Or are those still manually created ExternalModel CRs? | (a) Reconciler handles llm-d models only. Third-party models manually created. (b) Reconciler creates ExternalModel for all InferenceProviders including `api_provider` backend kind. | (a) for MVP. Third-party models are admin-configured with real API keys and provider-specific settings. |
| OQ-5 | **ACM/GitOps topology: what exactly is replicated?** Need to enumerate the CRDs and resources that ACM distributes. | MaaSSubscription, MaaSAuthPolicy, MaasTenantConfig, AITenant, ExternalModel, MaaSModelRef — but ExternalModel and MaaSModelRef are auto-created by the external model reconciler. Are they replicated FROM hubs, or created independently on each ingress cluster? | Created independently by each ingress cluster's external model reconciler (each ingress cluster sees the full mesh via SWIM). ACM replicates admin-managed CRDs only (subscriptions, auth policies, tenant config). |

---

## Sources

- Grid Operator CRDs: `praxis-proxy/grid/operator/src/crd/`
  (`inference_provider.rs`, `grid_network.rs`, `grid_site.rs`)
- Grid CRDT state: `praxis-proxy/grid/crdt/src/grid_state.rs`
- Grid scoring: `praxis-proxy/grid/scoring/src/backend.rs`
- Grid SWIM: `praxis-proxy/grid/swim/src/`
- MaaS ExternalModel: `models-as-a-service/maas-controller/api/maas/v1alpha1/externalmodel_types.go`
- MaaS MaaSModelRef: `models-as-a-service/maas-controller/api/maas/v1alpha1/maasmodelref_types.go`
- MaaS MaaSSubscription: `models-as-a-service/maas-controller/api/maas/v1alpha1/maassubscription_types.go`
- Auth design: `spare-notes/grid-mvp/auth-and-ratelimit.md`
- Signals contract: `spare-notes/grid-mvp/Ai-Grid-Signals-Contract.md`
- Deployment scenarios: `spare-notes/grid-mvp/deployment-scenarios.md`
