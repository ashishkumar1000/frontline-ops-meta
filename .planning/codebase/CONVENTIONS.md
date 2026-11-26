# Coding Conventions

**Analysis Date:** 2026-11-27

> **Important context:** This repository is a **specification-only repo** — there is no application
> source code, no lint/format tooling configured, and no CI yet (see `TESTING.md`). The conventions
> that govern work in this repo are **documentation conventions**: they define how the build-ready
> spec documents are written, named, cross-referenced, and kept consistent. When implementation
> begins, the *binding* engineering standards that future code must follow are already fixed in
> `docs/_context/engineering-decisions.md` and `docs/_context/conventions.md` — this file records
> both layers. The two canonical sources win over anything written here; this file summarizes them
> for planning purposes.

## Governing Sources (read before writing any document or code)

| Source | Authority | Path |
|---|---|---|
| Shared Conventions | Mandatory for every module doc; fixes tech stack, entities, API/event formats, state machines, diagram format | `docs/_context/conventions.md` |
| Engineering Decisions Register | **Binding**, platform-wide; where a doc is silent, this register's answer is the answer | `docs/_context/engineering-decisions.md` |
| Source baseline (v2.0 PRD/FSD extract) | Single source of truth for scope, entities, states, events, KPIs, terminology (modules 01–13) | `docs/_context/source-spec.md` |
| Modules addendum | Scope, entity map, shortcodes for modules 14–25 | `docs/_context/new-modules-addendum.md` |
| Shared-reference rules | What belongs in `_context/` vs. module folders; "module docs assume this folder and never repeat it" | `docs/_context/README.md` |

## Naming Patterns

**Files and directories:**
- Top-level docs use a zero-padded numeric prefix + hyphenated name: `docs/00-PRD.md`,
  `docs/01-FSD.md`, `docs/02-Epics-and-Stories.md`.
- Module folders are `NN-kebab-case-slug`: `docs/modules/01-identity-organization/` …
  `docs/modules/25-field-service-vendor-scheduling/`. The two-digit number is the module's stable ID.
- Every module folder has exactly the same five-file shape (snake_case filenames):
  `README.md` (plain-language overview), `functional-spec.md`, `high-level-design.md`,
  `low-level-design.md`, `architecture-diagram.md`. Do not add differently-named spec files.
- Shared material goes only in `docs/_context/` (`conventions.md`, `engineering-decisions.md`,
  `source-spec.md`, `new-modules-addendum.md`) — never duplicated into module folders.
- Root-level `functional-spec_1.md`, `high-level-design.md`, `low-level-design_1.md`,
  `architecture-diagram_1.md` are **legacy drafts of Module 03** kept for history only; canonical
  versions live in `docs/modules/03-forms-workflow-builder/`. Do not edit the root drafts.

**Requirement and traceability IDs (mandatory format):**
- Functional requirements: `FR-<SHORTCODE>-###`, three digits, e.g. `FR-FORM-001` in
  `docs/modules/03-forms-workflow-builder/functional-spec.md`. Shortcodes are fixed:
  baseline modules use names like `FORM`, `WORK`; gap modules 14–25 use
  `WFS`, `TA`, `CMMS`, `IOT`, `ENG`, `INV`, `EHS`, `HACCP`, `GAM`, `LAUNCH`, `MKT`, `FSVC`
  (defined in `docs/_context/new-modules-addendum.md`, "Numbering & file convention").
- Non-functional requirements: `NFR-<SHORTCODE>-##`, e.g. `NFR-FORM-04` (offline schema render ≤300ms).
- Stories: `EPIC-xx.S-yy` (e.g. `EPIC-01.S-01` in `docs/02-Epics-and-Stories.md`); every FR must be
  traceable to at least one story via these IDs.
- Module READMEs carry a two-row identity table: **Module code** (e.g. `FORM`), **Owning service**
  (e.g. `Form Service`), **Epics** (e.g. `EPIC-04`) — see `docs/modules/03-forms-workflow-builder/README.md`.

**Identifiers used inside spec content (when implementation starts, these become code naming rules):**
- Canonical domain entities are PascalCase and must not be renamed (extend only): `Tenant`,
  `OrganizationNode`, `User`, `UserScope`, `Role`, `Template`, `TemplateVersion`, `Submission`,
  `WorkItem`, `Issue`, `Evidence`, `KnowledgeDocument`, `AgentRun`, `AgentToolExecution`,
  `AuditLog`, `Asset`, `Course`/`LearningAssignment`, `Integration`, `WebhookSubscription`,
  `NotificationPreference` (`docs/_context/conventions.md`, "Domain Model").
- Database tables/columns are snake_case (e.g. `templates`, `template_versions`, `submissions`,
  `submission_answers`, `client_uuid`, `tenant_id`, `schema_hash` — see
  `docs/modules/03-forms-workflow-builder/low-level-design.md` §2).
- API error codes are SCREAMING_SNAKE_CASE with a stable enumerated set, e.g. `SCOPE_DENIED`,
  `DRAFT_EDIT_CONFLICT`, `IDEMPOTENCY_KEY_REUSED_DIFFERENT_PAYLOAD`, `CHAIN_HOP_LIMIT_EXCEEDED`.
- Kafka event types are dotted lowercase with versioned suffixes on breaking change:
  `audit.submitted`, `shift.published`, `submission.created.v2`.
- Lifecycle states are SCREAMING_SNAKE_CASE: WorkItem `DRAFT → SCHEDULED → AVAILABLE → IN_PROGRESS →
  SUBMITTED → PENDING_REVIEW → APPROVED → VERIFIED`; Issue and TemplateVersion machines are fixed
  verbatim in `docs/_context/conventions.md` ("State machines") — never invent new terminal states.
- Agent autonomy levels: `L0 Observe`, `L1 Recommend`, `L2 Execute low-risk`, `L3 Execute with
  approval`, `L4 Prohibited`.

## Code Style

**Formatting (documentation, in force today):**
- Plain, precise engineering prose and Markdown tables; minimal marketing language
  (`docs/_context/conventions.md`, "Tone & format").
- Numbered section headings (`## 1. Purpose and Scope`, `### 1.1 Purpose`) in every functional-spec
  and LLD; module-level requirements live under numbered sections.
- Catalog/index content is expressed as Markdown tables with a fixed column shape, e.g. the module
  catalog in `docs/modules/README.md` (`| Module | What it does | README |`), the doc map in
  `docs/README.md`, and the `_context` index in `docs/_context/README.md`.
- **No placeholders.** Never write "TBD"/"to be decided" — make an explicit engineering decision and
  state it; where genuinely open, mark `Decision needed:` with a recommended default
  (`docs/_context/conventions.md`). Former "Open Questions/Assumptions" sections have been rewritten
  as **"Resolved Decisions"** (see `docs/modules/03-forms-workflow-builder/functional-spec.md` §11).

**Linting (planned; binding for future code, from `docs/_context/engineering-decisions.md` §5/§10):**
- Kotlin (backend services): official Kotlin style guide via `ktlint` (format) + `detekt` (static
  analysis), hard CI gates. Spring Boot 3.x, Java 21 LTS.
- TypeScript (React Native mobile + Next.js web admin): `strict: true` in `tsconfig.json`;
  ESLint (`airbnb-typescript` base + a platform ruleset, e.g. banning direct network calls outside
  the designated sync layer) + Prettier, enforced in CI.
- Commit convention: Conventional Commits (`feat:`, `fix:`, `refactor:`) driving per-service
  changelogs. Branching: trunk-based, short-lived branches (<3 days), releases are tags off `main`.
- Merge gate: minimum 1 approving review + green CI including the contract-diff check
  (`engineering-decisions.md` §5/§10).
- No lint config exists in this repo today; the only package manifest is
  `.claude/package.json` (`{"type":"commonjs"}`, part of the GSD tooling, not an app).

## Cross-Reference Organization (the "imports" of this repo)

**Order of authority when documents disagree:**
1. `docs/_context/engineering-decisions.md` (binding register — a module doc never contradicts it)
2. `docs/_context/conventions.md` (house rules)
3. `docs/_context/source-spec.md` (baseline scope/entities — extend, never contradict)
4. `docs/_context/new-modules-addendum.md` (modules 14–25 scope and integration rules)
5. The individual module document

**Link style:**
- Documents cross-link with relative Markdown links, e.g. `[`docs/00-PRD.md`](docs/00-PRD.md)` from
  `README.md`, and `[README](./01-identity-organization/README.md)` within `docs/modules/README.md`.
- Normative references cite by name and section, e.g. "per conventions.md", "source-spec §18.3",
  "FR-FORM-045", "see low-level-design.md §2" — requirement IDs are the join key across documents.
- Reading order for implementers is fixed in `README.md` ("Implementing from these docs? Read in
  this order"): PRD → engineering-decisions → FSD → backlog → module docs (README → functional-spec
  → HLD → LLD, with `architecture-diagram.md` as visual companion).

**No-duplication rule:** cross-cutting material lives once in `docs/_context/`; module docs cover
only module-specific content and "assume this folder and never repeat it"
(`docs/_context/README.md`, "The rules").

## Error Handling

**Documentation pattern (in force):** every module's `low-level-design.md` ends with an **Error
Handling Table** with the fixed four-column shape — `| Error condition | Detection point | Response |
Recovery |` (example: `docs/modules/03-forms-workflow-builder/low-level-design.md` §11). These tables
are normative: `engineering-decisions.md` §7 states "those tables are the test plan, not just
documentation," and every documented error code must have an automated test once code exists (§11,
Definition of Done item 3).

**API error contract (fixed across all modules, `docs/_context/conventions.md`):**
- Envelope: `{ "error": { "code": "SCOPE_DENIED", "message": "...", "trace_id": "..." } }` with a
  stable enumerated `code`.
- Optimistic concurrency: `ETag`/`version`; conflicting writes return `409 Conflict` with a
  machine-readable `conflict` payload.
- Validation failures use `422` with a specific code (e.g. `SCHEMA_VALIDATION_FAILED`);
  expected-retry duplicates return `200` with `duplicate: true`, never an error.
- Status-code semantics follow the per-endpoint tables in each module LLD §5 (e.g.
  `docs/modules/03-forms-workflow-builder/low-level-design.md` §5.10 "Error Codes Summary").

**Domain rules that recur in error handling:**
- Idempotency-Key header required on all unsafe mutations where retries are possible (mobile sync,
  webhooks, agent tool execution); duplicates keyed on client-generated UUIDs (`client_uuid`).
- Submissions are never silently discarded — cross-record validation failures persist with
  `validation_status = FAILED_CROSS_RECORD` and route to a review queue.
- Evidence is never mutated; corrections are new linked records.

## Logging

**Framework:** none exists (no code). **Spec-pinned standard** (`engineering-decisions.md` §9):
- OpenTelemetry SDK in every service and client; traces/metrics/logs correlated by `trace_id`.
- Structured JSON logs; every line carries `trace_id` / `tenant_id` / `request_id`.
- Mobile crash reporting (Sentry or Crashlytics) mandatory from the first mobile release.
- Every mutating endpoint emits a canonical domain event in the envelope of `source-spec.md` §18.4
  (`event_id`, `event_type`, `version`, `tenant_id`, `occurred_at`, `actor_id`, `resource_id`,
  `trace_id`, `data`).

## Comments (Diagrams and Narration)

- **Mermaid-only diagrams.** All architecture/sequence/ER/state diagrams are ```mermaid code blocks —
  no images, no ASCII art (`docs/_context/conventions.md`, "Diagram conventions"). Renderers used:
  `flowchart TD`/`graph LR` (component/data flow), `sequenceDiagram` (request/event flows),
  `erDiagram` (data models), `stateDiagram-v2` (lifecycles).
- Diagram placement: each module's `architecture-diagram.md` contains (1) a component/container
  diagram, (2) at least one sequence diagram for the primary flow, (3) an ER diagram of owned
  entities, (4) a state diagram if the module owns a lifecycle — with 2–4 sentences of narration per
  diagram and **no large prose blocks** (prose belongs in the other three files). Reference example:
  `docs/modules/03-forms-workflow-builder/architecture-diagram.md` (7 Mermaid blocks).
- LLDs additionally embed ER diagrams and sequence diagrams inline next to the relevant schema and
  flow sections (e.g. `docs/modules/03-forms-workflow-builder/low-level-design.md` §2.2, §6).

## Function Design (Pseudocode and Algorithm Specifications)

**Size/depth expectations are fixed per document type** (`docs/_context/conventions.md`, "Document
depth expectations per file"):
- `functional-spec.md`: numbered FRs with business rules; UX states incl. empty/loading/offline/
  error/permission-denied; field-level behavior; validation & edge cases; module NFRs; dependencies.
- `high-level-design.md`: responsibilities/boundaries, component diagram, key decisions & trade-offs,
  data ownership (owned vs. referenced entities), external interfaces (APIs/events in/out),
  scalability & multi-tenancy, failure modes & degradation, justified tech choices.
- `low-level-design.md`: concrete DB schema (tables + `erDiagram`), concrete REST endpoints (method,
  path, request/response JSON examples, status/error codes), sequence diagrams for the 3–5 most
  important flows, pseudocode for non-trivial logic (recurrence expansion, SLA escalation, conflict
  merge, risk scoring), caching strategy, concurrency control, background jobs, error table.
- `architecture-diagram.md`: focused Mermaid diagrams only (see Comments above).

**Parameters/returns (future API code):** base path `/api/v1/`; OAuth2/OIDC bearer auth; cursor
pagination `?cursor=...&limit=...` → `{ "items": [...], "next_cursor": "..." }`; Idempotency-Key on
unsafe methods; every table/query/event/storage key carries `tenant_id`, enforced server-side, with
`OrganizationNode` scope filtering as the second axis (`docs/_context/conventions.md`,
"Multi-tenancy & Scoping Rule").

## Module Design (Repository and Package Layout — binding for future code)

Per `docs/_context/engineering-decisions.md` §4 (**polyrepo**, one repo per deployable):
- `platform-contracts` — OpenAPI 3.1 specs + JSON Schema for every Kafka event + generated
  Kotlin/TypeScript SDKs. All consumers depend on published artifacts, never hand-copied types.
- One repo per microservice (`svc-work-orchestration`, `svc-forms`, `svc-identity`, …).
- `mobile-app` — single React Native app (modules are feature boundaries, not separate installs).
- `web-admin` — one Next.js/React app for all admin surfaces.
- `infra` — Terraform + Helm charts + ArgoCD Application manifests (GitOps).

**Reuse rules (gap modules must not duplicate engines):** everything executable is a `WorkItem`
(module 02 owns execution); one approval mechanism (Work Orchestration's `ApprovalRoute`); Evidence
(05) never mutated; "policy before AI" for agents (10) and Visual AI (09); server is source of truth;
no module owns payroll — see the recurrence list in `docs/modules/README.md` ("How the modules fit
together"). New shared primitives/extensions (e.g. `WorkItem.assignee_type=VENDOR`, new WorkItem
types) are declared once in `docs/_context/new-modules-addendum.md` and reused consistently.

**Pinned tech stack — do not invent alternatives** (`docs/_context/conventions.md`, "Target Tech
Stack"): React Native (TypeScript) + WatermelonDB offline; React + TypeScript web; Kotlin + Spring
Boot microservices (gRPC internal, REST external); PostgreSQL with `tenant_id` + RLS on every table;
S3-compatible object storage (per-tenant prefix, KMS); Kafka (tenant-partitioned by
`tenant_id:aggregate_id`); Temporal.io; OpenSearch + pgvector; Redis; LiteLLM-style model gateway +
OPA/Rego policy; Kubernetes/Terraform/Helm/Argo Rollouts; OpenTelemetry.

---

*Convention analysis: 2026-11-27*
