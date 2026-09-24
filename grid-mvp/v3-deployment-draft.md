# AI Grid v3 Demo — Deployment Layout

```mermaid
graph TD
    subgraph Clients
        C1["Consumer<br/><i>curl / SDK</i><br/>Bearer JWT"]
    end

    subgraph DNS / Ingress
        R["OpenShift Route<br/><b>v3-gateway</b><br/>TLS edge"]
    end

    C1 -->|HTTPS| R

    subgraph "Namespace: ai-grid-v3"

        subgraph "Gateway Pod"
            GW["<b>v3-gateway</b><br/>praxis-ai-gateway:geo-labels-token-v1<br/>ai@45c67142 · 0.6.0 core<br/>:8080 gateway · :9901 admin"]
            CM["ConfigMap: v3-praxis-config<br/><i>praxis.yaml + policy.yaml</i>"]
            CM -.->|mounted| GW
        end

        R -->|":8080"| GW

        subgraph "Filter Chain (in-process)"
            F1["policy<br/><i>JWT auth (HS256)</i><br/><i>require(authenticated)</i>"]
            F2["model_to_header<br/><i>X-Gateway-Model-Name</i>"]
            F3["intelligent_route<br/><i>match_claims: grid_region → region label</i><br/><i>candidates: site-us, site-eu, site-uk</i>"]
            F4["token_rate_limit<br/><i>key: authenticated_subject</i><br/><i>sliding_window 1h / 100 tokens</i>"]
            F5["token_count<br/><i>provider: openai</i>"]
            F6["load_balancer<br/><i>3 clusters</i>"]
            F1 --> F2 --> F3 --> F4 --> F5 --> F6
        end

        subgraph "Mock Backends"
            US["<b>mock-us</b><br/>Deployment · 1 replica<br/>grid-mock-providers:v0.1.1<br/>:8080 · OpenAI dialect<br/><i>region: us-east-1</i>"]
            EU["<b>mock-eu</b><br/>Deployment · 1 replica<br/>grid-mock-providers:v0.1.1<br/>:8080 · OpenAI dialect<br/><i>region: eu-west-1</i>"]
            UK["<b>mock-uk</b><br/>Deployment · 1 replica<br/>grid-mock-providers:v0.1.1<br/>:8080 · OpenAI dialect<br/><i>region: eu-west-2</i>"]
        end

        F6 -->|"cluster: site-us"| US
        F6 -->|"cluster: site-eu"| EU
        F6 -->|"cluster: site-uk"| UK
    end

    subgraph "JWT Minting (offline)"
        MINT["mint-region-tokens.py<br/><i>HS256 · demo secret</i><br/><i>claims: sub, grid_region, grid_rate, grid_burst</i>"]
    end

    MINT -.->|"token"| C1
```

## Request Flow

1. **Consumer** sends `POST /v1/chat/completions` with `Authorization: Bearer <JWT>`
2. **policy** — verifies HS256 JWT (issuer, audience, signature). 401 if invalid
3. **model_to_header** — extracts `model` from body, promotes to `X-Gateway-Model-Name`
4. **intelligent_route** — reads `grid_region` claim from JWT, matches against candidate labels. Picks the cluster whose `region` label matches. 403 if no region claim (fail-closed)
5. **token_rate_limit** — checks per-subject sliding window budget. 429 if exhausted
6. **token_count** — reads token usage from upstream response (OpenAI format)
7. **load_balancer** — forwards to the selected cluster endpoint (mock-us/eu/uk)

## Tokens

| Name | Subject | grid_region | Routes to |
|------|---------|-------------|-----------|
| US | us-user | us-east-1 | mock-us (site-us) |
| EU | eu-user | eu-west-1 | mock-eu (site-eu) |
| UK | uk-user | eu-west-2 | mock-uk (site-uk) |
| NONE | noregion-user | *(absent)* | 403 fail-closed |
