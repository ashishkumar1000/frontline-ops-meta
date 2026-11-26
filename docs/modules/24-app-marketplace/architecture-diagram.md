# Architecture Diagrams — App Marketplace & Template Catalog

Module: `24-app-marketplace` (shortcode `MKT`)

## 1. Component Diagram

```mermaid
flowchart TD
    subgraph Publisher["Publisher / Reviewer"]
        PubUI[Publisher Workspace]
        RevUI[Review Queue UI]
    end

    subgraph MKT["Marketplace Service"]
        API[Marketplace API]
        Pipeline[Publishing Pipeline]
        Orchestrator[Install Orchestrator]
        MktDB[(Marketplace Postgres)]
        Idx[(OpenSearch catalog index)]
        Cache[(Redis catalog cache)]
    end

    subgraph Tenant["Tenant Admin"]
        CatalogUI[Catalog Browse / Install Preview UI]
    end

    subgraph Downstream["Materialization Targets"]
        Forms[Forms & Workflow Builder]
        IntHub[Integration Hub]
        Identity[Identity & Org Service]
    end

    PubUI --> API
    RevUI --> API
    API --> Pipeline --> MktDB
    Pipeline -. sandbox dry-run .-> Orchestrator

    CatalogUI --> API
    API --> Cache
    API --> Idx
    API --> Orchestrator
    Orchestrator -->|copy content: create Template/TemplateVersion| Forms
    Orchestrator -->|copy content: create Integration shell| IntHub
    Orchestrator -->|check scope/plan/role| Identity
    Orchestrator --> MktDB
```

**Narrative**: Publishers and Reviewers operate entirely against the Marketplace Service's own
store — nothing they do touches tenant data until a Reviewer's sandbox dry-run, which reuses the
same orchestrator code path a real tenant install uses but targets a disposable sandbox tenant.
Tenant admins browse a cached/indexed read model and, on install, the Install Orchestrator is the
**only** component in this module that calls another module's write API — and it always calls it as
a normal tenant-scoped client, creating brand-new rows in Forms & Workflow Builder / Integration
Hub rather than writing into their tables directly or handing out a reference to marketplace-owned
content.

## 2. Sequence Diagram — Tenant Install (Copy-on-Install Emphasis)

```mermaid
sequenceDiagram
    participant Admin
    participant MKT as Marketplace Service
    participant Forms as Forms & Workflow Builder
    participant IntHub as Integration Hub

    Admin->>MKT: Install "QSR Opening Pack" v2.1.0
    MKT->>MKT: Read app_version_bundle_items (read-only, shared across ALL tenants)
    MKT->>Forms: Create NEW Template + TemplateVersion (deep copy of bundle schema)
    Forms-->>MKT: tenant-owned template_version_id = tv_701a
    MKT->>Forms: Create NEW Workflow + Schedule (referencing tv_701a)
    Forms-->>MKT: workflow_id, schedule_id
    MKT->>IntHub: Create NEW Integration shell (NOT_CONFIGURED)
    IntHub-->>MKT: integration_id
    MKT->>MKT: Write app_installations row: tenant-scoped pointer to the NEW ids only
    MKT-->>Admin: Install complete — content is now yours to edit
```

**Narrative**: The bundle definition read from `app_version_bundle_items` is never mutated and never
referenced by foreign key from the newly created `tenant-owned` rows — every downstream create call
returns a brand-new id that belongs entirely to the installing tenant. If a second tenant installs
the same `AppVersion` a minute later, this exact sequence repeats independently and produces a
second, completely distinct set of ids with no relationship to the first tenant's copy other than
both having originated from the same immutable bundle content.

## 3. Sequence Diagram — Update Available, Opt-In Upgrade

```mermaid
sequenceDiagram
    participant Publisher
    participant MKT as Marketplace Service
    participant Admin as Tenant Admin
    participant Forms as Forms & Workflow Builder

    Publisher->>MKT: List AppVersion 2.1.0
    MKT->>MKT: Flag update_available=true on every ACTIVE installation of this listing
    MKT-->>Admin: "Update available" badge + changelog
    Admin->>MKT: Request upgrade (opt-in)
    MKT->>MKT: Diff installed content vs. tenant's current (possibly customized) content
    alt Tenant has NOT customized
        MKT->>Forms: Publish new TemplateVersion (auto-adopt)
    else Tenant HAS customized
        MKT-->>Admin: Manual reconciliation view (adopt / keep / parallel, per item)
        Admin->>MKT: Reconciliation choices
        MKT->>Forms: Apply only the chosen changes per item
    end
    MKT->>MKT: Update app_installations.current_app_version_id; retain full history
    MKT-->>Admin: Upgrade complete — no silent overwrite occurred
```

**Narrative**: An update is never pushed; it is only ever pulled, and pulling it never overwrites a
tenant's customization without an explicit per-item choice when divergence is detected. Declining to
upgrade is a fully supported, indefinite state.

## 4. ER Diagram — Owned Entities

```mermaid
erDiagram
    PUBLISHERS ||--o{ APP_LISTINGS : owns
    APP_LISTINGS ||--o{ APP_VERSIONS : "has versions"
    APP_VERSIONS ||--o{ APP_VERSION_BUNDLE_ITEMS : bundles
    APP_LISTINGS ||--o{ APP_INSTALLATIONS : "installed as (per tenant)"
    APP_VERSIONS ||--o{ APP_INSTALLATIONS : "current version of"
    APP_INSTALLATIONS ||--o{ APP_INSTALLATION_HISTORY : records
    APP_LISTINGS ||--o{ APP_REVIEWS : "rated by (per tenant)"

    PUBLISHERS {
        uuid id PK
        string verification_status
    }
    APP_LISTINGS {
        uuid id PK
        uuid publisher_id FK
        string category
        numeric average_rating
    }
    APP_VERSIONS {
        uuid id PK
        uuid app_listing_id FK
        string version
        string status
    }
    APP_VERSION_BUNDLE_ITEMS {
        uuid id PK
        uuid app_version_id FK
        string item_type
        jsonb definition
    }
    APP_INSTALLATIONS {
        uuid id PK
        uuid tenant_id
        uuid app_listing_id FK
        uuid current_app_version_id FK
        uuid_array materialized_template_version_ids
        string status
    }
    APP_INSTALLATION_HISTORY {
        uuid id PK
        uuid app_installation_id FK
        string action
    }
    APP_REVIEWS {
        uuid id PK
        uuid tenant_id
        uuid app_listing_id FK
        int rating
    }
```

**Narrative**: Note that `APP_INSTALLATIONS.materialized_template_version_ids` is a plain array of
ids, not a foreign key relationship into a tenant's `template_versions` table — deliberately, since
that table lives in the Forms & Workflow Builder's own schema (a different bounded context) and this
module never joins across that boundary at the database level, only via API calls (see
high-level-design.md §6).

## 5. State Diagram — AppVersion Publishing Lifecycle

```mermaid
stateDiagram-v2
    [*] --> DRAFT: publisher creates version
    DRAFT --> SUBMITTED: publisher submits (FR-MKT-011 checks pass)
    SUBMITTED --> IN_REVIEW: reviewer claims from queue
    IN_REVIEW --> APPROVED: reviewer approves
    IN_REVIEW --> REJECTED: reviewer rejects (reason_code required)
    IN_REVIEW --> DRAFT: reviewer requests changes
    SUBMITTED --> REJECTED: auto-reject on failed automated security/structural lint
    APPROVED --> LISTED: auto-list or manual "list now"
    LISTED --> DEPRECATED: publisher or platform operator deprecates
    REJECTED --> DRAFT: publisher revises and creates a new draft attempt
    DEPRECATED --> [*]
    REJECTED --> [*]

    note right of LISTED
        Only LISTED versions are visible/
        installable in tenant-facing catalog
        search (FR-MKT-022).
    end note
    note right of DEPRECATED
        Existing installations keep working;
        new installs are blocked and redirected
        to a newer LISTED version if one exists
        (FR-MKT-014).
    end note
```

**Narrative**: The lifecycle mirrors the platform's existing `TemplateVersion` state-machine style
(immutable-once-published, explicit deprecation rather than deletion) so publishers and reviewers
experience a governance model consistent with the rest of the platform. The mandatory `IN_REVIEW`
human gate before `APPROVED`/`LISTED` is the module's primary defense against malicious or broken
submissions reaching tenants (functional-spec.md §6), and it is structurally impossible to skip —
there is no transition from `SUBMITTED` or `DRAFT` directly to `LISTED`.
