# Analytics & Control Tower — High-Level Design

Module corresponds to **EPIC-12 (Control Tower)** and **EPIC-15 (Analytics & ML)**. Implements the
"Analytics Platform" component from source-spec §16.3 and the "ANALYTICS PIPELINE" branch of the
logical architecture (§16.1).

## 1. Module Responsibilities

The Analytics Platform is a single logical module (may be split into ingestion, warehouse-compute,
and serving deployables per §7) with five responsibilities:

1. **Ingestion** — durable, exactly-once-effective capture of every relevant domain event
   (Event Backbone topics, §18.3) and every relevant Postgres table change (CDC via Debezium, per
   conventions.md) from every other module's transactional store.
2. **Warehouse / lakehouse** — a bronze/silver/gold medallion-layered lakehouse holding raw,
   cleaned/conformed, and business-aggregated representations of that data.
3. **Metrics semantic layer** — a single governed definition of every KPI (formula, grain,
   segmentation dimensions, versioning) that all consumers (dashboards, exports, BI tools, the
   Executive Insights Agent) query through, so no two surfaces can compute VOER differently.
4. **Dashboard API** — a query/read API serving the three Control Tower views and drill-down
   navigation, with caching and freshness-tier awareness.
5. **ML forecasting / anomaly-detection service** — a separate compute path (batch + light online
   scoring) producing location risk scores, KPI forecasts, and repeated-failure-pattern detections,
   feeding back into the gold layer as first-class forecast facts.

This module **owns derived and aggregate facts only**. It never becomes the system of record for
operational state — a WorkItem, Issue, Submission, or AgentRun's authoritative state always lives
in its owning service's transactional database. If the warehouse and the source service ever
disagree, the source service is correct and the warehouse is stale (a freshness problem, not a
data-integrity dispute).

## 2. Component Diagram

```mermaid
flowchart TB
    subgraph Sources["Source Services (systems of record)"]
        WS[Work Service - Postgres]
        IS[Issue/CAPA Service - Postgres]
        VA[Visual AI Service - Postgres]
        AR[Agent Runtime - Postgres]
        OS[Org/Identity Service - Postgres]
        CS[Comms/Learning Services - Postgres]
    end

    subgraph Backbone["Event Backbone (Kafka)"]
        EVT[Domain event topics\ntask.* issue.* agent.* message.* training.*]
        CDCT[Debezium CDC topics\nper-table change streams]
    end

    subgraph Ingestion["Streaming Ingestion Layer"]
        SC[Stream Consumers\nKafka Connect / Spark Structured Streaming]
        SCH[Schema Registry\n+ contract validation]
    end

    subgraph Lakehouse["Lakehouse (Databricks/Snowflake/BigQuery-class)"]
        BRONZE[(Bronze\nraw append-only,\nas-received)]
        SILVER[(Silver\ncleaned, conformed,\ndeduped, typed)]
        GOLD[(Gold\nstar-schema facts +\ndims, pre-aggregated)]
    end

    subgraph Semantic["Metrics Semantic Layer"]
        MD[metric_definitions registry]
        MQ[Metrics Query Engine\n(dbt-metrics / Cube-class)]
    end

    subgraph MLPipeline["ML Forecasting & Anomaly Pipeline"]
        FE[Feature Store]
        TRAIN[Batch Training Jobs\n(risk model, forecast model)]
        SCORE[Scoring Jobs\n(daily risk score, forecast refresh)]
        ANOM[Anomaly Detector\n(streaming + batch)]
    end

    subgraph Serving["Serving Layer"]
        DAPI[Dashboard / Metrics Query API]
        CACHE[(Query Result Cache - Redis)]
        RTPATH[Near-Real-Time Path\n(direct read of Issue/Work service replicas\nfor critical counters)]
    end

    subgraph Consumers["Consumers"]
        UI[Control Tower UI\n(Executive/Regional/Store)]
        EIA[Executive Insights Agent\n(Agent Runtime)]
        BI[BI Connectors\n(Power BI/Tableau/Snowflake)]
        EXP[Scheduled Export Jobs]
    end

    WS -- CDC --> CDCT
    IS -- CDC --> CDCT
    VA -- CDC --> CDCT
    AR -- CDC --> CDCT
    OS -- CDC --> CDCT
    CS -- CDC --> CDCT
    WS -- domain events --> EVT
    IS -- domain events --> EVT
    VA -- domain events --> EVT
    AR -- domain events --> EVT
    CS -- domain events --> EVT

    EVT --> SC
    CDCT --> SC
    SC --> SCH
    SCH --> BRONZE
    BRONZE --> SILVER
    SILVER --> GOLD
    GOLD --> MD
    MD --> MQ
    GOLD --> FE
    FE --> TRAIN
    TRAIN --> SCORE
    SILVER --> ANOM
    SCORE --> GOLD
    ANOM --> GOLD

    MQ --> DAPI
    GOLD --> DAPI
    DAPI --> CACHE
    IS -.direct read.-> RTPATH
    WS -.direct read.-> RTPATH
    RTPATH --> DAPI

    DAPI --> UI
    DAPI --> EIA
    DAPI --> BI
    DAPI --> EXP
```

The near-real-time path (dashed lines) is an intentional bypass of the medallion pipeline for the
small set of counters that cannot tolerate T-1 latency (critical exception count, unacknowledged
critical issues — functional-spec.md §3.4, §8.4). It reads directly from operational service read
replicas/APIs and is reconciled nightly against the warehouse's view of the same Issues (§8
background jobs) so the two paths never silently diverge for long.

## 3. Data Ownership

| Data | Owner | This module's relationship |
|---|---|---|
| WorkItem, Issue, Submission, Evidence, AgentRun, OrganizationNode, User (operational/source state) | Respective owning module | Consumed via CDC/events; read-only, never mutated here |
| Bronze layer (raw copies of source events/rows) | Analytics Platform | Owned; append-only, retained per tenant data-retention policy, replayable to rebuild silver/gold |
| Silver layer (conformed, deduped, typed) | Analytics Platform | Owned; still one-to-one-ish with source entities, not yet business-metric shaped |
| Gold layer (star-schema facts/dims, pre-aggregated) | Analytics Platform | Owned; the only layer other modules/consumers may query |
| `metric_definitions` (KPI formulas, versions) | Analytics Platform | Owned; Operations Admin edits through this module's config API, never a raw SQL edit |
| `risk_forecasts`, anomaly flags, repeated-failure-pattern clusters | Analytics Platform (ML pipeline) | Owned derived data; written back to gold, exposed via API, never written into source services directly (a manager must explicitly accept a recommended action for it to become a real WorkItem/Issue) |
| Dashboard layout/widget configuration per role | Analytics Platform | Owned; tenant-configurable via Operations Admin console |

## 4. External Interfaces

### 4.1 Consumed

- **Event Backbone** — subscribes to effectively all canonical events in source-spec §18.3
  (`task.*`, `audit.*`, `issue.*`, `training.*`, `message.*`, `agent.*`), each in the standard
  envelope (§18.4), partitioned by `tenant_id:aggregate_id`.
- **CDC streams (Debezium)** — one logical stream per source table that feeds a gold fact or
  dimension: `work_items`, `issues`, `submissions`, `agent_runs`, `agent_tool_executions`,
  `organization_nodes`, `user_scopes`, `evidence` (metadata only), `visual_ai_results`.
- **Organization/Identity scope API** — read of effective-dated `OrganizationNode`/`UserScope` for
  dashboard scope enforcement (not just warehouse rollup attribution) — every dashboard query is
  additionally filtered server-side by the requesting principal's current scope, resolved live
  against the Authorization Service, not solely from a warehouse snapshot, so a scope change takes
  effect immediately rather than waiting for next day's batch.

### 4.2 Exposed

- **Metrics Query API** (`/api/v1/analytics/...`) — the single query surface for KPI values,
  segmentation, drill-down, control-tower summaries, and risk forecasts. Consumed by: Control Tower
  UI (web admin BFF), the Executive Insights Agent (as a typed retrieval tool per source-spec
  §20.2-style tool contract, e.g. `get_kpi_snapshot`, `get_location_risk`), BI connectors
  (Integration Hub), and scheduled export jobs.
- **`analytics.threshold_breached` event** — new canonical event this module produces onto the
  Event Backbone when a configured KPI threshold is crossed, consumed by Notification Hub and
  optionally by Agent Runtime.
- **Export/report generation API** — on-demand and scheduled report definitions and their run
  history.

## 5. Scalability

- **Volume**: platform NFR target is 100k locations / 1M+ users / millions of work items per day
  (source-spec §23). At that scale, computing VOER-class KPIs on-the-fly per dashboard load with
  raw joins across WorkItem/Issue/Submission tables is not viable — hence the medallion
  pre-aggregation strategy: gold-layer fact tables are pre-aggregated at `location x
  workflow_type/failure_category x day` grain by nightly (and, for near-real-time counters,
  micro-batch) rollup jobs, and a dashboard query is a scan/rollup over pre-aggregated rows, not a
  join over raw operational events.
- **Pre-aggregation vs. on-the-fly trade-off**: pre-aggregation is used for every KPI whose grain is
  known in advance (the fixed §14.2 KPI set) — this gives predictable, cache-friendly query latency
  at the cost of a fixed set of supported cuts. On-the-fly querying against the silver layer is
  reserved for drill-down's final hops (individual WorkItem/Issue lists), where result sets are
  small (bounded by pagination) and the query is naturally scoped to a single location/short period
  by the time the user has drilled that far, so a live silver-layer query is cheap. This module does
  not support fully ad-hoc arbitrary-dimension pivoting over raw facts in the base product (flagged
  as a possible later phase, functional-spec.md §11); the semantic layer's fixed metric+dimension
  catalog is the deliberate boundary that keeps pre-aggregation tractable.
- **Streaming ingestion throughput**: sized for millions of events/day sustained, with burst
  headroom (e.g. end-of-month campaign completion spikes) via Kafka partition scaling and
  autoscaled stream-consumer compute; ingestion never blocks source-service writes (it is
  asynchronous and downstream of CDC/event publish, per conventions.md event-driven principle).
- **Query concurrency**: dashboard API targets 10,000 concurrent sessions (functional-spec.md §9)
  via a Redis-backed query result cache keyed by `(tenant_id, scope, metric_set, filters,
  definition_version)` with TTL tied to each metric's freshness tier (§8), so repeated views of the
  same enterprise/region rollup (common — many managers viewing similar cuts) are served from cache,
  not recomputed per request.

## 6. Multi-Tenancy

- Every bronze/silver/gold table carries `tenant_id`, consistent with the platform-wide rule
  (conventions.md). In the lakehouse this is enforced two ways: (1) logical row-level security —
  warehouse-native RLS/row-access policies (Snowflake row access policies / Databricks Unity
  Catalog row filters / BigQuery row-level security, whichever platform is selected) scoped to the
  querying service principal's tenant context, applied even though the Dashboard API also filters
  by tenant at the query-construction layer (defense in depth, mirroring the platform's "tenant
  identifier is not sufficient by itself" rule, source-spec §21.1); (2) large enterprise tenants
  that are isolated to a dedicated Postgres schema/cluster at the operational layer (per
  conventions.md tenant-tier sharding) are also given a dedicated warehouse schema/database, so
  their CDC stream never lands in a shared gold table with other tenants' data.
- The ML pipeline trains **per-tenant or per-tenant-cohort models**, never a single cross-tenant
  model trained on pooled raw data without an explicit tenant opt-in to a shared-benchmarking
  program (a future, clearly consent-gated feature, out of scope here). Feature Store rows,
  training datasets, and model artifacts are tenant-tagged and access-controlled identically to raw
  data.
- Query result cache keys include `tenant_id` as the leading key segment; a cache key collision
  across tenants is structurally impossible (not just unlikely) because the key is namespaced by
  tenant before any other dimension.

## 7. Failure Modes and Degradation

| Failure mode | Detection | Degradation strategy |
|---|---|---|
| CDC lag (Debezium connector behind source WAL) | Consumer lag metric per connector vs. threshold alert | Gold-layer rollup jobs skip a partition until its bronze data is confirmed complete; dashboard shows partial-data state (functional-spec.md FR-ANLY-016) rather than blocking; near-real-time direct-read path (§2) is unaffected since it bypasses CDC entirely for the counters that matter most under this failure |
| Streaming ingestion job failure/crash | Consumer group health check, dead-letter topic volume | Automatic restart with checkpoint resume (Kafka offset commit only after successful bronze write, §9 concurrency); backlog drains via replay from Kafka retention window (sized to cover the maximum expected outage) with no data loss as long as replay happens within topic retention |
| Gold-layer rollup job failure (partial run) | Job orchestrator (e.g. Databricks Jobs/Airflow) failure alert | Job is idempotent per partition (re-running a partition overwrites deterministically, not additively); last successful gold snapshot continues to serve reads with its `as_of` timestamp shown in the stale-data banner (FR-ANLY-017) until the next successful run |
| ML model staleness (retrain job failed or skipped past cadence) | Model registry tracks `last_trained_at` vs. expected cadence (functional-spec.md §9) | Forecasts/risk scores continue to serve from the last successfully trained model, visibly labeled "Forecast based on model last trained {date}" (FR-ANLY-019); a model past a hard staleness ceiling (e.g. 3x expected cadence) is suppressed from the UI entirely rather than shown indefinitely stale, falling back to a simple rules-based risk band (no ML) so the Control Tower never goes fully blank |
| Near-real-time path and warehouse disagree (reconciliation drift) | Nightly reconciliation job diff (§8) | Warehouse (gold) figure is authoritative for T-1 KPIs; the direct-read counters are authoritative for their own narrow near-real-time purpose; a persistent drift beyond tolerance raises an internal data-quality alert (Operations Admin visible) rather than silently averaging the two |
| Warehouse platform outage (vendor-side) | Health check / query error rate | Dashboard API serves last cached result from Redis with an extended-staleness banner; scheduled exports queue and retry rather than fail silently; near-real-time counters (bypassing the warehouse) remain available, so the Control Tower degrades to "critical counters only" rather than going fully dark |

## 8. Technology Choices (justified)

- **Medallion architecture (bronze/silver/gold)** — chosen over a single flat warehouse layer
  because the platform's source data is heterogeneous (relational CDC rows + semi-structured domain
  events + eventual ML feature outputs) and arrives with varying quality/completeness. Bronze
  preserves a faithful, replayable raw record (critical for reprocessing after a bug in
  transformation logic, and for the audit/traceability expectations that run throughout this
  platform per source-spec P7/P8); silver is where schema conformance, deduplication (idempotent on
  `event_id`/CDC LSN), and type normalization happen once, so every downstream consumer works from
  the same cleaned representation; gold is shaped specifically around the KPI set and Control Tower
  query patterns (star schema, pre-aggregated), which is what makes the dashboard API's latency
  targets achievable at 100k-location scale.
- **A governed metrics semantic layer instead of ad-hoc SQL per dashboard** — with 8+ KPIs each
  needing exact, auditable definitions, multiple consumers (three Control Tower views, BI exports,
  the Executive Insights Agent, scheduled reports), and versioning requirements (functional-spec.md
  §8.1), letting each consumer write its own SQL against gold tables would inevitably produce subtly
  different VOER numbers on different screens — a credibility-destroying failure mode for an
  executive-facing "single source of truth" product. A semantic layer (dbt-metrics/Cube-class tooling)
  centralizes the formula + grain + versioning once, and every consumer (including the agent, which
  must never be allowed to "compute its own KPI" from raw retrieval) calls the same metric
  resolution logic.
- **CDC (Debezium) as the primary ingestion path for operational entities** rather than each
  service double-writing to both its transactional DB and the warehouse — keeps source services
  simple and consistent (single write, per conventions.md), decouples analytics availability from
  a synchronous dependency in the write path of any operational service, and gives natural
  replayability (re-run ingestion from a WAL position) if transformation logic needs to be fixed.
- **Separate ML forecasting pipeline (feature store + batch training/scoring) rather than embedding
  forecasting logic inside the dashboard query path** — training and heavy scoring are
  computationally different workloads from interactive dashboard queries and must scale/fail
  independently; keeping them as a distinct pipeline that writes results back into the gold layer as
  ordinary facts (`risk_forecasts`) means the Dashboard API's read path never has to know whether a
  number came from a rollup job or a model — it is just another gold fact with a freshness
  timestamp and confidence field, keeping the serving layer simple and uniform.
- **Redis query result cache with freshness-tier-aligned TTL** — reuses the platform's standard
  cache technology (conventions.md) rather than introducing a separate caching product, and ties TTL
  directly to each metric's declared freshness SLA (functional-spec.md §9) so the cache can never
  serve data staler than the product's own stated promise.
