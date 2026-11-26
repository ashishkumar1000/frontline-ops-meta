# Shared Conventions — Frontline Operations & Agentic Ops Platform

Every module document (Functional Spec, HLD, LLD, Architecture Diagram) MUST stay consistent with
this file. Read `source-spec.md` in this same folder first — it is the extracted content of the
original v2.0 PRD/FSD baseline and is the single source of truth for scope, entities, states,
events, KPIs and terminology. Do not contradict it; extend and deepen it.

This is a **work-class, mobile-first B2B SaaS product** for frontline operations (retail, QSR,
hospitality, facilities, logistics, manufacturing floor, healthcare-adjacent site ops). Assume the
audience is engineering leadership and senior engineers who will build directly from these docs —
write at build-ready depth: concrete schemas, concrete endpoints, concrete field lists, concrete
edge cases. Do not write vague placeholders like "TBD" or "to be decided" — make a reasonable,
explicit engineering decision and state it. Where genuinely open, mark as `Decision needed:` with a
recommended default so work is not blocked.

## Target Tech Stack (use these consistently across all modules — do not invent alternatives)

- **Mobile (frontline app):** React Native (TypeScript) for iOS + Android from one codebase, with
  native modules (Kotlin/Swift) for camera pipeline, barcode/QR scanning, background sync, and
  biometric auth where the RN bridge is insufficient. Local embedded DB: WatermelonDB (SQLite-backed,
  reactive, built for offline-first sync). Background work: WorkManager (Android) / BGTaskScheduler
  (iOS). Push: FCM (Android) + APNs (iOS) via a unified Notification Hub.
- **Web Admin / Control Tower:** React + TypeScript, component library with design tokens, BFF-driven
  GraphQL or REST aggregation layer.
- **Backend services:** Kotlin + Spring Boot (JVM) microservices, one deployable per bounded context
  (see Core Technical Components table in source spec). Internal sync calls via gRPC; external via
  REST/JSON behind the API Gateway.
- **API Gateway / BFF:** Node.js (TypeScript) or Spring Cloud Gateway — tenant resolution, authN
  termination, rate limiting, request shaping per client (mobile vs web).
- **Primary datastore:** PostgreSQL (Aurora/Cloud SQL-class managed service), **row-level multi-
  tenancy** via `tenant_id` on every table + Postgres Row-Level Security (RLS) policies as a defense-
  in-depth backstop behind application-layer scoping. Large enterprise tenants may be isolated to a
  dedicated schema or cluster (tenant-tier based sharding).
- **Object storage:** S3-compatible (AWS S3 / GCS) for evidence media, with per-tenant prefix +
  KMS-encrypted, signed-URL access.
- **Event backbone:** Apache Kafka (or managed equivalent — MSK/Confluent). Topics are
  tenant-partitioned by key `tenant_id:aggregate_id` for ordering.
- **Workflow / scheduling engine:** Temporal.io for durable, versioned workflow execution
  (schedules, SLA timers, escalation ladders, CAPA state machines).
- **Search & retrieval:** OpenSearch for operational/document search; pgvector (or a managed vector
  store) for RAG embeddings behind the Knowledge Hub and Agent Runtime retrieval tool.
- **Cache:** Redis (permission cache, session cache, rate-limit counters, idempotency keys).
- **Analytics:** CDC (Debezium) from Postgres + event backbone into a lakehouse
  (Snowflake/BigQuery/Databricks-class) via a streaming ingestion layer; BI via the warehouse.
- **Agent Runtime / AI:** Model-provider-agnostic gateway (LiteLLM-style abstraction) fronting
  OpenAI/Anthropic/Azure/Bedrock models; typed tool-calling; policy engine implemented with OPA
  (Open Policy Agent) / Rego for declarative, testable authorization and agent-action policy.
- **IaC / Deploy:** Kubernetes (EKS/GKE-class), Terraform, per-service Helm charts, progressive
  delivery (Argo Rollouts) tied to the Release Path in the PRD (feature flag → internal → beta →
  pilot → 5%→25%→50%→100%).
- **Observability:** OpenTelemetry traces/metrics/logs → a unified backend (Grafana/Datadog-class),
  correlated by `trace_id` per the Request Context fields in the source spec.

## Multi-tenancy & Scoping Rule (applies to every module)

Every table, cache key, search index document, vector, event, queue message and object-storage key
carries `tenant_id`. Every query path enforces `tenant_id` server-side — never trust a client-
supplied tenant context except where explicitly permitted for support tooling (and even then it is
logged and scoped). `OrganizationNode` scoping (arbitrary-depth tree, effective-dated) is the second
enforcement axis: a principal's `UserScope` rows define which node subtrees they can act on, and
every read/write API additionally filters by scope, not tenant alone.

## Domain Model — canonical entities (extend, do not rename)

Use exactly these entity names when a module references core objects (full field lists live in the
FSD, `01-FSD.md`): `Tenant`, `OrganizationNode`, `User`, `UserScope`, `Role`, `Template`,
`TemplateVersion`, `Submission`, `WorkItem`, `Issue`, `Evidence`, `KnowledgeDocument`, `AgentRun`,
`AgentToolExecution`, `AuditLog`, `Asset`, `Course` / `LearningAssignment`, `Integration`,
`WebhookSubscription`, `NotificationPreference`.

## API conventions (must match across modules)

- Base path `/api/v1/`, OAuth2/OIDC bearer auth, tenant resolved from the authenticated principal.
- Idempotency-Key header required on all unsafe (POST/PUT/PATCH) mutation endpoints where retries
  are possible (mobile sync, webhook handlers, agent tool execution).
- Optimistic concurrency via `ETag` / `version` field; conflicting writes return `409 Conflict`
  with a machine-readable `conflict` payload.
- Cursor-based pagination: `?cursor=...&limit=...` → `{ "items": [...], "next_cursor": "..." }`.
- Errors: `{ "error": { "code": "SCOPE_DENIED", "message": "...", "trace_id": "..." } }` with a
  stable enumerated `code`.
- Every mutating endpoint emits a canonical domain event (see source spec §18.3) onto the Event
  Backbone in the envelope shape from §18.4.

## State machines (reuse verbatim; do not invent new terminal states)

- **WorkItem:** `DRAFT → SCHEDULED → AVAILABLE → IN_PROGRESS → SUBMITTED → PENDING_REVIEW →
  APPROVED → VERIFIED`; side branches `IN_PROGRESS → BLOCKED`, `PENDING_REVIEW → REJECTED →
  IN_PROGRESS`, `AVAILABLE/IN_PROGRESS → OVERDUE → ESCALATED`.
- **Issue:** `OPEN → TRIAGED → ASSIGNED → INVESTIGATING → CORRECTIVE_ACTION → VERIFICATION →
  RESOLVED → CLOSED`, with `RESOLVED/VERIFICATION → REOPENED → INVESTIGATING` on failed
  effectiveness check.
- **TemplateVersion:** `DRAFT → REVIEW → APPROVED → PUBLISHED → SUPERSEDED → RETIRED`.
- **Agent autonomy levels:** `L0 Observe`, `L1 Recommend`, `L2 Execute low-risk`, `L3 Execute with
  approval`, `L4 Prohibited`.

## Diagram conventions

All architecture/sequence/ER diagrams are written as **Mermaid** code blocks (```mermaid) so they
render directly in GitHub/GitLab/VS Code/most doc tools without external image generation.
Use `flowchart TD`/`graph LR` for component & data-flow diagrams, `sequenceDiagram` for
request/response and event flows, `erDiagram` for data models, `stateDiagram-v2` for lifecycle
state machines, and `C4Context`/`flowchart` conventions for deployment views.

## Document depth expectations per file (per module)

- **functional-spec.md** — Purpose & scope of the module; in-scope/out-of-scope; personas served;
  detailed functional requirements (numbered, e.g. `FR-<MOD>-001`) with business rules; UX states
  incl. empty/loading/offline/error/permission-denied; field-level behavior for any forms/objects
  owned by the module; validation & edge cases; non-functional requirements specific to the module;
  dependencies on other modules; open questions/assumptions.
- **high-level-design.md** — Module responsibilities & boundaries; component diagram (mermaid);
  how it fits the platform architecture; key design decisions & trade-offs; data ownership (which
  entities this service owns vs. references); external interfaces (APIs/events consumed & produced);
  scalability & multi-tenancy approach; failure modes & degradation strategy; technology choices
  specific to the module (justified).
- **low-level-design.md** — Concrete DB schema (tables, columns, types, keys, indexes) as both a
  table and an `erDiagram`; concrete REST endpoint list with method, path, request/response JSON
  examples, status codes, error codes; sequence diagrams (mermaid) for the 3–5 most important flows;
  algorithms/pseudocode for non-trivial logic (e.g. scheduling recurrence expansion, SLA escalation
  timers, conflict resolution merge, risk scoring); caching strategy; concurrency control; background
  jobs/workers and their triggers; detailed error handling table.
- **architecture-diagram.md** — A small set of focused Mermaid diagrams for this module only:
  (1) component/container diagram, (2) at least one sequence diagram for the primary flow,
  (3) an ER diagram of the module's owned entities, (4) a state diagram if the module owns a
  lifecycle. Add 2–4 sentences of narration per diagram — no large blocks of prose (that belongs in
  the other three files).

## Traceability

Every functional requirement should be traceable to one or more Epics/Stories in
`02-Epics-and-Stories.md` — use the `EPIC-xx` IDs from the source spec's Appendix E and reference
story IDs as `EPIC-xx.S-yy`.

## Tone & format

Write in plain, precise, engineering-document prose and tables — minimal marketing language. Use
numbered requirement IDs so items are traceable. Do not repeat the entire original PRD text
verbatim; synthesize, extend and go deeper, referencing back to source-spec.md concepts by name.
