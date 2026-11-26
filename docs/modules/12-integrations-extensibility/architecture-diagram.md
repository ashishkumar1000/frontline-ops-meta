# Integration Hub — Architecture Diagrams

Module: `12-integrations-extensibility`
Epic: EPIC-13 (Integration Hub)

## 1. Component Diagram

```mermaid
flowchart TD
    subgraph AdminSurface["Web Admin / Control Tower"]
        Wizard["Connector Setup Wizard"]
        WebhookUI["Webhook Subscription Manager"]
        BatchUI["Batch Job Console"]
        HealthUI["Connection Health Dashboard"]
    end

    GW["API Gateway / BFF"]

    subgraph Hub["Integration Hub Service"]
        Registry["Connector Registry & Config Store"]
        WebhookEngine["Webhook Delivery Engine"]
        BatchOrch["Batch Job Orchestrator"]
        InboundAPI["Inbound REST/SDK Surface"]
    end

    subgraph Adapters["Per-Connector Adapter Workers"]
        HR["HR Adapters"]
        ERP["ERP Adapters"]
        Data["Data Adapters"]
        BI["BI Adapters"]
        Collab["Collaboration Adapters"]
        Ops["Operational-System Adapters"]
        SFTP["SFTP Worker"]
    end

    Vault["Central Secret Manager"]
    EB["Event Backbone (Kafka)"]
    Ext["External Systems\n(HR/ERP/Data/BI/Chat/POS/SFTP/Webhook Consumer/Partner App)"]

    AdminSurface --> GW --> Hub
    Registry <--> Vault
    WebhookEngine <--> Vault
    InboundAPI <--> Vault
    Registry --> Adapters
    BatchOrch --> SFTP
    Adapters <--> Ext
    EB --> WebhookEngine
    EB --> Adapters
    WebhookEngine --> Ext
    GW --> InboundAPI
    InboundAPI <--> Ext
```

**Narration**: The Integration Hub Service sits behind the API Gateway alongside every other
domain service, but it is unusual in fanning both directions: inward from the admin UI for
configuration, and outward through per-category adapter workers to the entire external system
landscape. The Connector Registry is the single source of configuration truth and the only
component that writes credential *references*; actual secrets always round-trip through the
Central Secret Manager. The Webhook Delivery Engine and every adapter worker consume the same
tenant-partitioned Event Backbone that the rest of the platform publishes to, making this module the
platform's largest single consumer of the canonical event catalog.

## 2. Sequence Diagram — Webhook Delivery with Retry

```mermaid
sequenceDiagram
    participant EB as Event Backbone
    participant WE as Webhook Delivery Engine
    participant Vault as Secret Vault
    participant C as Consumer Endpoint

    EB->>WE: domain event (e.g. issue.resolved)
    WE->>Vault: fetch signing secret
    WE->>WE: HMAC-SHA256 sign payload
    WE->>C: POST signed webhook
    alt success
        C-->>WE: 200 OK
        WE->>WE: mark DELIVERED
    else failure
        C-->>WE: non-2xx / timeout
        WE->>WE: schedule retry (exponential backoff + jitter)
        loop until max_attempts
            WE->>C: retry POST
        end
        WE->>WE: mark DEAD_LETTERED, alert owner
    end
```

**Narration**: Every delivery is attempted, and on failure the engine schedules a retry using
exponential backoff with jitter rather than hammering a failing consumer. After the configured
attempt budget is exhausted the delivery is dead-lettered and the subscription owner is alerted; the
delivery remains permanently replayable from the delivery log, and — critically — a replay reuses
the same `(subscription_id, event_id)` delivery row rather than creating a new one, so retries and
manual replays can never cause duplicate downstream processing on a correctly implemented consumer.

## 3. Sequence Diagram — SFTP/CSV Batch Ingestion

```mermaid
sequenceDiagram
    participant Sched as Batch Scheduler
    participant SFTP as SFTP Worker
    participant Stage as Staging Bucket
    participant Orch as Batch Orchestrator
    participant Domain as Domain Service

    Sched->>SFTP: dispatch pickup
    SFTP->>Stage: stage file + checksum
    Orch->>Stage: read staged file
    Orch->>Orch: structural validation
    loop each row
        Orch->>Orch: field mapping + transform
        Orch->>Domain: upsert (idempotent)
    end
    Orch->>Orch: record accepted/rejected counts + error report
```

**Narration**: Batch ingestion decouples the slow, fragile SFTP transfer from validation and
ingestion by staging the file in object storage first. Structural validation can fail the whole
file early and cheaply; row-level validation isolates bad records so one malformed row never blocks
the rest of the file, and every downstream write is idempotent on the record's external reference so
a re-run of the same file is always safe.

## 4. ER Diagram — Owned Entities

```mermaid
erDiagram
    INTEGRATIONS ||--o{ WEBHOOK_SUBSCRIPTIONS : "may drive"
    INTEGRATIONS ||--o{ CONNECTOR_FIELD_MAPPINGS : "has"
    INTEGRATIONS ||--o{ BATCH_JOBS : "may drive"
    INTEGRATIONS ||--o| CONNECTOR_CREDENTIALS : "references"
    WEBHOOK_SUBSCRIPTIONS ||--o{ WEBHOOK_DELIVERIES : "generates"
    BATCH_JOBS ||--o{ BATCH_JOB_RUNS : "executes as"

    INTEGRATIONS {
        uuid id PK
        uuid tenant_id
        text connector_type
        text category
        text state
        uuid owner_user_id
        uuid scope_node_id
        text credential_ref
    }
    WEBHOOK_SUBSCRIPTIONS {
        uuid id PK
        uuid tenant_id
        uuid integration_id FK
        text target_url
        text_array event_types
        text signing_secret_ref
        text status
    }
    WEBHOOK_DELIVERIES {
        uuid id PK
        uuid subscription_id FK
        uuid event_id
        text status
        int attempt_count
        timestamptz next_retry_at
    }
    CONNECTOR_FIELD_MAPPINGS {
        uuid id PK
        uuid integration_id FK
        text entity_type
        text source_field
        text target_field
        boolean is_required
    }
    BATCH_JOBS {
        uuid id PK
        uuid integration_id FK
        text direction
        text transfer_mechanism
        text schedule_cron
    }
    BATCH_JOB_RUNS {
        uuid id PK
        uuid batch_job_id FK
        text status
        int row_count_accepted
        int row_count_rejected
    }
    CONNECTOR_CREDENTIALS {
        uuid id PK
        uuid integration_id FK
        text vault_ref
        text rotation_state
    }
```

**Narration**: `Integration` is the aggregate root for a configured connector, referencing (not
owning) its credential by vault path through `ConnectorCredential`. A single `Integration` can drive
zero or more `WebhookSubscription`s and/or `BatchJob`s depending on its transfer mechanism, and owns
its own versioned `ConnectorFieldMapping` set. `WebhookDelivery` rows are uniquely keyed on
`(subscription_id, event_id)` — the schema-level guarantee behind idempotent replay.

## 5. State Diagram — WebhookDelivery Lifecycle

```mermaid
stateDiagram-v2
    [*] --> PENDING: event matched to subscription
    PENDING --> DELIVERING: delivery attempt dispatched
    DELIVERING --> DELIVERED: 2xx response
    DELIVERING --> FAILED: non-2xx / timeout
    FAILED --> RETRY_SCHEDULED: attempts remain in budget
    FAILED --> DEAD_LETTERED: attempt budget exhausted
    RETRY_SCHEDULED --> DELIVERING: next_retry_at reached
    DEAD_LETTERED --> RETRY_SCHEDULED: manual replay (same delivery row)
    DELIVERED --> [*]
    DEAD_LETTERED --> [*]: retained for audit/replay indefinitely
```

**Narration**: A delivery cycles through `PENDING → DELIVERING` on every attempt, and `FAILED`
attempts either get rescheduled or, once the retry budget is exhausted, become `DEAD_LETTERED`.
Dead-lettered deliveries are not deleted — they remain in the delivery log indefinitely and can be
moved back to `RETRY_SCHEDULED` by an explicit manual replay, which updates this same row (never
inserts a new one), which is what keeps replay safe against duplicate downstream processing.
