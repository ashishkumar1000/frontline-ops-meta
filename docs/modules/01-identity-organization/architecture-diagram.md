# Identity & Organization — Architecture Diagrams

## 1. Component / Container Diagram

```mermaid
flowchart TB
    subgraph Clients
        WEB[Web Admin / Control Tower]
        MOBILE[Mobile App]
        OTHERSVC[Other Platform Services]
    end

    GATEWAY[API Gateway / BFF]

    subgraph IdentityOrgModule[Identity & Organization Module]
        IDENTITY[Identity Service\nusers, SSO, SCIM, sessions]
        ORG[Organization Service\ntenants, org graph, effective dating]
        AUTHZ[Authorization Service\nroles, scopes, policy eval, Effective Access]
    end

    PG[(PostgreSQL)]
    REDIS[(Redis)]
    KAFKA[[Event Backbone]]
    IDP[External IdP]
    OPA[OPA Policy Engine]

    WEB --> GATEWAY
    MOBILE --> GATEWAY
    OTHERSVC -->|gRPC CheckPermission| AUTHZ
    GATEWAY --> IDENTITY
    GATEWAY --> ORG
    GATEWAY --> AUTHZ
    IDENTITY <--> IDP
    IDENTITY --> PG
    ORG --> PG
    AUTHZ --> PG
    AUTHZ <--> REDIS
    AUTHZ --> OPA
    IDENTITY --> KAFKA
    ORG --> KAFKA
    AUTHZ --> KAFKA
```

This module is three logical services — Identity, Organization, Authorization — deployed initially
as one bounded-context service and split independently only when scale or ownership demands it. The
Gateway routes all administrative and login traffic here, while every other platform service reaches
the Authorization Service directly over internal gRPC for permission checks on every request it
serves; no other service queries this module's tables directly. Redis and OPA sit behind the
Authorization Service as a performance cache and a declarative attribute-policy evaluator,
respectively, keeping the sub-15ms permission-check latency budget achievable at 1M-user scale.

## 2. Sequence Diagram — Permission Check (primary flow)

```mermaid
sequenceDiagram
    participant SvcX as Any Module Service
    participant AuthZ as Authorization Service
    participant Redis
    participant PG as Postgres

    SvcX->>AuthZ: CheckPermission(user, resource, action, node_key)
    AuthZ->>Redis: GET perm:{tenant}:{user}:{node}
    alt cache hit
        Redis-->>AuthZ: decision set
    else cache miss
        AuthZ->>PG: resolve UserScope via closure-table ancestor lookup + Role permissions
        PG-->>AuthZ: scopes + role permissions
        AuthZ->>AuthZ: deny-overrides-allow resolution
        AuthZ->>Redis: SET perm:... (TTL 5s)
    end
    AuthZ-->>SvcX: Decision{allow/deny, provenance}
```

Every permission check first tries the Redis cache, populated on demand, before touching Postgres.
The Authorization Service resolves the caller's ancestor chain through the closure table (an O(depth)
lookup), applies role permissions from each contributing `UserScope`, and returns a decision plus its
provenance — the same code path the Effective Access preview reuses so the two can never diverge.

## 3. ER Diagram — Owned Entities

```mermaid
erDiagram
    TENANTS ||--o{ ORGANIZATION_NODES : contains
    TENANTS ||--o{ USERS : contains
    TENANTS ||--o{ ROLES : "custom roles"
    ORGANIZATION_NODES ||--o{ ORGANIZATION_NODE_CLOSURE : "ancestor/descendant"
    USERS ||--o{ USER_SCOPES : holds
    ROLES ||--o{ USER_SCOPES : "grants via"
    ORGANIZATION_NODES ||--o{ USER_SCOPES : "scoped at"
    USERS ||--o{ SESSIONS : has
    TENANTS ||--o{ IDENTITY_PROVIDER_CONFIGS : configures
    TENANTS ||--o{ SCIM_PROVISIONING_BINDINGS : configures

    TENANTS {
        uuid id PK
        text status
        text plan
    }
    ORGANIZATION_NODES {
        uuid id PK
        uuid node_key
        ltree path
        timestamptz valid_from
        timestamptz valid_to
    }
    USERS {
        uuid id PK
        text identity_ref
        text status
    }
    ROLES {
        uuid id PK
        jsonb permissions
    }
    USER_SCOPES {
        uuid id PK
        text effect
        timestamptz valid_from
        timestamptz valid_to
    }
```

`OrganizationNode` is effective-dated (one row per version, keyed by a stable `node_key`), which is
why it is drawn distinctly from the current-state-only `OrganizationNodeClosure` performance index.
`UserScope` is the join between `User`, `Role`, and a node, and is itself effective-dated so that
"who could access what, when" remains answerable after any role or reorg change.

## 4. OrganizationNode Effective-Dating Timeline (conceptual)

This module has no single lifecycle state machine (unlike WorkItem, Issue, or TemplateVersion) —
instead, each logical node accumulates a sequence of non-overlapping, effective-dated versions.
The diagram below shows one `SITE` node's version history across a reorg and a rename.

```mermaid
flowchart LR
    subgraph V1["Version 1 — valid 2025-01-01 → 2026-09-30"]
        A1["node_key: 9a11\nparent: Region West (root.42.317)\nname: Store 4471"]
    end
    subgraph V2["Version 2 — valid 2026-10-01 → 2026-10-31"]
        A2["node_key: 9a11\nparent: Region West (root.42.317)\nname: Store 4471 - Denver"]
    end
    subgraph V3["Version 3 — valid 2026-11-01 → open"]
        A3["node_key: 9a11\nparent: Region North (root.55)\nname: Store 4471 - Denver Tech Center"]
    end

    A1 -->|"rename effective_at=2026-10-01"| A2
    A2 -->|"reorg move effective_at=2026-11-01"| A3

    F1["Fact: Audit submitted 2026-08-15"] -.attributed to.-> A1
    F2["Fact: Audit submitted 2026-10-15"] -.attributed to.-> A2
    F3["Fact: Audit submitted 2026-11-15"] -.attributed to.-> A3
```

Each version's `[valid_from, valid_to)` window is disjoint and contiguous for the same `node_key`.
Historical facts (audits, submissions, issues) resolve their organizational attribution by matching
their own `occurred_at` timestamp into the version window that contained it at the time, which is
what keeps a 2026-08-15 audit permanently attributed to Region West even after the store moves to
Region North on 2026-11-01 — satisfying the platform's "Reorganization" acceptance test without any
special-case logic in the reporting layer.
