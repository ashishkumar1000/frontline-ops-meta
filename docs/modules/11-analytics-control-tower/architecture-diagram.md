# Analytics & Control Tower — Architecture Diagrams

Module corresponds to **EPIC-12 (Control Tower)** and **EPIC-15 (Analytics & ML)**. Focused diagram
set only; narrative detail lives in high-level-design.md and low-level-design.md.

## 1. Component Diagram

```mermaid
flowchart TB
    subgraph Sources["Source Services"]
        WS[Work Service]
        IS[Issue/CAPA Service]
        VA[Visual AI Service]
        AR[Agent Runtime]
        OS[Org/Identity Service]
    end

    subgraph Backbone["Event Backbone (Kafka) + CDC (Debezium)"]
        EVT[Domain events + CDC topics]
    end

    subgraph Ingestion["Streaming Ingestion"]
        SC[Stream Consumers + Schema Registry]
    end

    subgraph Lakehouse["Lakehouse"]
        BRONZE[(Bronze)]
        SILVER[(Silver)]
        GOLD[(Gold - star schema)]
    end

    subgraph Semantic["Metrics Semantic Layer"]
        MQ[Metric Definitions + Query Engine]
    end

    subgraph ML["ML Forecasting / Anomaly Pipeline"]
        TRAIN[Training Jobs]
        SCORE[Scoring Jobs]
        ANOM[Anomaly Detector]
    end

    subgraph Serving["Serving Layer"]
        DAPI[Dashboard / Metrics Query API]
        CACHE[(Redis Cache)]
        RT[Near-Real-Time Direct-Read Path]
    end

    subgraph Consumers
        UI[Control Tower UI]
        EIA[Executive Insights Agent]
        BI[BI Connectors / Exports]
    end

    Sources -- CDC + events --> EVT
    EVT --> SC --> BRONZE --> SILVER --> GOLD
    GOLD --> MQ --> DAPI
    SILVER --> ANOM --> GOLD
    GOLD --> TRAIN --> SCORE --> GOLD
    DAPI --> CACHE
    IS -.direct read.-> RT
    WS -.direct read.-> RT
    RT --> DAPI
    DAPI --> UI
    DAPI --> EIA
    DAPI --> BI
```

This is the module's single most important structural fact: two ingestion paths feed one serving
layer. The medallion path (CDC/events -> bronze -> silver -> gold) computes every T-1 KPI and all
ML outputs; a narrow near-real-time bypass reads operational services directly for the handful of
counters (critical exceptions, unacknowledged critical issues) that cannot tolerate batch latency.
Both converge at the Dashboard API so every consumer — UI, the Executive Insights Agent, and BI
connectors — sees one coherent response shape regardless of which path served a given field.

## 2. Sequence Diagram — Ingestion to Dashboard

```mermaid
sequenceDiagram
    participant Src as Source Service (Postgres)
    participant Debezium
    participant Kafka
    participant Ingest as Ingestion Consumer
    participant Gold as Gold Rollup Job
    participant API as Dashboard API
    participant UI as Control Tower UI

    Src->>Debezium: row change (WAL)
    Debezium->>Kafka: CDC record
    Kafka->>Ingest: consume + checkpoint offset
    Ingest->>Gold: bronze/silver conform (async)
    Note over Gold: nightly rollup, idempotent per (tenant, date_key)
    UI->>API: GET metric query
    API->>Gold: read pre-aggregated fact
    Gold-->>API: rows + as_of timestamp
    API-->>UI: metric response (value, data_completeness, freshness_tier)
```

An event's journey from a source-service write to a dashboard-visible number crosses an
asynchronous boundary at every arrow after the initial WAL capture. The API always attaches
`as_of` and `data_completeness` so the UI can distinguish "fully current" from "computed over a
partial period" without guessing from missing data alone.

## 3. Sequence Diagram — Drill-Down from Enterprise KPI to Locations/Issues

```mermaid
sequenceDiagram
    participant Exec as Executive
    participant API as Dashboard API
    participant Auth as Authorization Service
    participant Gold

    Exec->>API: Enterprise VOER anomaly noticed on Control Tower
    Exec->>API: drilldown(scope=ENTERPRISE, next=region)
    API->>Auth: scope check
    Auth-->>API: authorized
    API->>Gold: query by region
    Gold-->>API: worst region identified
    API-->>Exec: region list, ranked by delta
    Exec->>API: drilldown(scope=region, next=location)
    API->>Gold: query by location
    Gold-->>API: worst location identified
    API-->>Exec: location list
    Exec->>API: drilldown(scope=location, next=work_item)
    API->>Gold: contributing WorkItems/Issues
    Gold-->>API: records with issue_id links
    API-->>Exec: hand-off to Issues/CAPA record view
```

Each hop is an independently scope-checked server query against the gold layer, never a
client-side re-filter of a broader result set already in the browser — this is what keeps
drill-down safe under the platform's tenant/scope isolation rules even as a user pivots from an
enterprise-wide view down to a single store's open issues.

## 4. ER Diagram — Gold-Layer Star Schema

```mermaid
erDiagram
    dim_location ||--o{ fact_work_item_completions : location_key
    dim_location ||--o{ fact_issues : location_key
    dim_location ||--o{ risk_forecasts : location_key
    dim_date ||--o{ fact_work_item_completions : date_key
    dim_date ||--o{ fact_issues : date_key_opened
    fact_issues }o--o| repeated_failure_patterns : pattern_cluster_id
    metric_definitions ||--o{ fact_work_item_completions : metric_definition_version_id

    dim_location {
        bigint location_key PK
        uuid tenant_id
        uuid location_id
        timestamptz valid_from
        timestamptz valid_to
    }
    dim_date {
        int date_key PK
        date calendar_date
    }
    fact_work_item_completions {
        bigint fact_key PK
        uuid tenant_id
        bigint location_key FK
        int date_key FK
        boolean is_verified
        boolean is_on_time
        uuid metric_definition_version_id FK
    }
    fact_issues {
        bigint fact_key PK
        uuid tenant_id
        bigint location_key FK
        text severity
        boolean repeat_flag
        uuid pattern_cluster_id FK
    }
    metric_definitions {
        uuid id PK
        text metric_code
        int version
        timestamptz effective_from
        timestamptz effective_to
    }
    risk_forecasts {
        uuid id PK
        bigint location_key FK
        text risk_band
        jsonb recommended_action
    }
    repeated_failure_patterns {
        uuid id PK
        text scope_type
        int occurrence_count
    }
```

The star schema is deliberately shallow: two conformed dimensions carry almost all rollup logic
(`dim_location` for the entire geography/hierarchy axis, `dim_date` for time), while
`metric_definitions` and `risk_forecasts` sit alongside the facts as first-class governed/derived
data rather than being bolted on as configuration outside the warehouse.

## 5. Data-Quality Flow — Medallion Bronze -> Silver -> Gold with Validation Gates

This module has no single object lifecycle (no `WorkItem`/`Issue`-style state machine of its own),
so in place of a state diagram this section documents the equivalent structural backbone: the
data-quality gates every record passes through as it moves from raw capture to a Control-Tower-
visible fact.

```mermaid
flowchart LR
    A[Raw CDC record / domain event] --> B{Schema Registry\ncontract valid?}
    B -- No --> B1[Dead-letter topic\n+ alert]
    B -- Yes --> C[Bronze: append-only raw write\nkeyed by event_id / table+pk+lsn]
    C --> D{Silver conformance:\ndedupe, type check,\nreferential sanity}
    D -- Fails referential check\ne.g. unknown location_id --> D1[Quarantine table\n+ data-quality alert,\nretried on next run]
    D -- Passes --> E[Silver: conformed row]
    E --> F{Gold eligibility:\nbelongs to a completed\nsource-of-truth state?}
    F -- No, e.g. WorkItem\nstill IN_PROGRESS --> F1[Held out of this run's\ngold aggregate;\nre-evaluated next run]
    F -- Yes --> G[Gold rollup job:\nidempotent per\n(tenant, date_key) partition]
    G --> H{Reconciliation check\nvs. near-real-time path\n(Issues only)}
    H -- Drift beyond tolerance --> H1[Internal data-quality alert\n(Operations Admin panel)]
    H -- Within tolerance --> I[Gold fact visible to\nDashboard API / Metrics layer]
```

Every gate either passes a record forward or routes it to an explicit remediation path (dead-letter,
quarantine, hold-for-next-run, or an internal alert) — a record is never silently dropped, and
nothing reaches the gold layer, and therefore the Control Tower UI, without having passed schema
validation, deduplication, and referential sanity checks first.
