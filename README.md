# AI Gateway — Design & Analysis

Collection of architecture designs, migration analyses, gap assessments,
and research for the AI Gateway platform (multi-cluster inference
routing, token management, and model governance).

## Structure

| Folder | Scope |
|--------|-------|
| [grid-mvp/](grid-mvp/) | Multi-cluster Grid deployment: architecture decisions, component gaps, deployment scenarios, SVG diagrams |
| [tokenomics/](tokenomics/) | Token rate limiting, metering, billing, and quota management: Praxis strategy research, demo analysis, migration plans |
| [pricetag/](pricetag/) | Pricetag dogfood: production gaps, key management migration (Pricetag → MaaS → Grid), product alignment |

## Diagrams

| Diagram | Preview |
|---------|---------|
| Multi-cluster Data Plane | [View](https://htmlpreview.github.io/?https://github.com/danielezonca/spare-notes/blob/main/grid-mvp/grid-dataplane.html) |
| Tenant Admin Flows | [View](https://htmlpreview.github.io/?https://github.com/danielezonca/spare-notes/blob/main/grid-mvp/grid-tenant-admin.html) |
| Tenant User Flows | [View](https://htmlpreview.github.io/?https://github.com/danielezonca/spare-notes/blob/main/grid-mvp/grid-tenant-user.html) |
