<!-- refreshed: 2026-11-27 -->
# Architecture

**Analysis Date:** 2026-11-27

## Critical Context: This Is a Specification-Only Repository

**There is no application source code in this repository.** No package manifests, no build configs,
no tests, no runtime entry points. The repo holds a **build-ready markdown documentation package**
(768 `.md` files) for the "Frontline Operations & Agentic Ops Platform" — a mobile-first B2B SaaS
product — plus AI engineering tooling (GSD + BMAD). The architecture described below is the
architecture **of the documentation package and tooling as it exists today**. The platform's
*target* runtime architecture (Kotlin/Spring Boot, Postgres RLS, Kafka, Temporal, React Native) is
**specified content** inside `docs/01-FSD.md` and `docs/_context/conventions.md` and is flagged as
such wherever referenced. Do not confuse the two.

## System Overview

```text
┌──────────────────────────────────────────────────────────────────────┐
│                     INDEX / NAVIGATION LAYER                          │
│  `README.md` (repo index)  ·  `CLAUDE.md` (AI routing rules)          │
│  `docs/README.md` (file-by-file doc index)                            │
└────────┬───────────────────────────────┬─────────────────────────────-┘
         │                               │
         ▼                               ▼
┌────────────────────────────┐  ┌───────────────────────────────────────┐
│   SHARED CONTEXT LAYER     │  │        PLATFORM SPEC LAYER            │
│   `docs/_context/`         │  │  `docs/00-PRD.md`    (703 lines)      │
│  conventions.md (stack)    │  │  `docs/01-FSD.md`    (1854 lines)     │
│  engineering-decisions.md  │  │  `docs/02-Epics-and-Stories.md`       │
│  (BINDING register)        │  │  (15 epics → 154 stories)             │
│  source-spec.md (baseline) │  │                                       │
│  new-modules-addendum.md   │  │                                       │
└────────┬───────────────────┘  └──────────────┬────────────────────────┘
         │  (every module doc assumes this,    │ (backlog traces to
         │   never repeats it)                 │  modules via EPIC-xx)
         ▼                                     ▼
┌──────────────────────────────────────────────────────────────────────┐
│                       MODULE SPEC LAYER                               │
│        `docs/modules/` — 25 folders × fixed 5-file set                │
│   `README.md` · `functional-spec.md` · `high-level-design.md`         │
│   `low-level-design.md` · `architecture-diagram.md`                   │
│   catalog: `docs/modules/README.md`                                   │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ consumed by
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    AI WORKFLOW TOOLING LAYER                          │
│  GSD (execution): `.claude/commands/gsd/` (67 cmds),                  │
│    `.claude/agents/` (33 agents), `.claude/get-shit-done/` (v1.42.3)  │
│  BMAD (planning): `.claude/skills/` + `.zcode/skills/` (36 skills),   │
│    `_bmad/` (config + scripts) · output: `_bmad-output/` (empty)      │
│  analysis output: `.planning/codebase/` (this document)               │
└──────────────────────────────────────────────────────────────────────┘
```

**Specified TARGET runtime architecture (documentation content, not repo reality):** a modular
monolith decomposed into independently deployable services along bounded-context lines, defined in
`docs/01-FSD.md` §1 with nine ADRs (§2, ADR-001…ADR-009) and a fixed tech stack in
`docs/_context/conventions.md` ("Target Tech Stack" section): React Native + WatermelonDB mobile,
React + TypeScript web admin, Kotlin/Spring Boot JVM services behind an API Gateway/BFF,
PostgreSQL with row-level multi-tenancy + RLS, Kafka event backbone, Temporal workflow engine,
Redis, S3, OpenSearch + pgvector, OPA/Rego policy engine, Debezium CDC → lakehouse analytics,
Kubernetes/Terraform/Argo Rollouts delivery. The **binding** polyrepo layout for implementation
(`platform-contracts`, one `svc-*` repo per service, `mobile-app`, `web-admin`, `infra`) is defined
in `docs/_context/engineering-decisions.md` §4 — no such repos exist yet.

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| Repo index | Top-level navigation; routing table "if you want X go to Y"; reading order for implementers | `README.md` |
| AI workflow routing | Mandatory BMAD-vs-GSD routing rules; declares `docs/` + `.planning/codebase/` as source of truth | `CLAUDE.md` |
| Doc index | File-by-file map of the package; module/epic mapping table; how-to-read order | `docs/README.md` |
| Master PRD | Vision, personas, mobile-first strategy, product principles, capability map, 25-module overviews, roadmap, metrics | `docs/00-PRD.md` |
| Platform FSD | Target architecture, ADRs, domain model, state machines, API/event contracts, offline sync protocol, AI governance, security, observability, NFRs | `docs/01-FSD.md` |
| Backlog | 15 epics decomposed into 154 build-ready user stories with acceptance criteria | `docs/02-Epics-and-Stories.md` |
| Shared conventions | Binding tech stack, tenant-scoping rule, canonical entities, API/event formats, state machines, Mermaid/diagram format, per-file depth expectations, requirement-ID traceability | `docs/_context/conventions.md` |
| Engineering decisions | BINDING register: OAuth 2.1/OIDC auth mechanics, OS support matrix, API versioning, polyrepo structure, CI/CD, environments, testing gates, DoD, EU AI Act posture | `docs/_context/engineering-decisions.md` |
| Source baseline | Extracted v2.0 PRD/FSD text; single source of truth for scope/terminology of modules 01–13 | `docs/_context/source-spec.md` |
| Modules addendum | Scope, entity map, and integration rules for competitive-gap modules 14–25 | `docs/_context/new-modules-addendum.md` |
| Module catalog | One-line description + README link for all 25 modules; cross-module integration patterns | `docs/modules/README.md` |
| Module folders | Build-ready per-module spec quartet + plain-language README (125 files, ~42.9k lines) | `docs/modules/NN-<name>/` |
| GSD execution layer | Slash commands, subagents, workflows, hooks, templates for the GSD execution loop | `.claude/commands/gsd/`, `.claude/agents/`, `.claude/get-shit-done/`, `.claude/hooks/` |
| BMAD planning layer | 36 BMAD skills (planning/discovery personas); duplicated identically for Claude and ZCode | `.claude/skills/`, `.zcode/skills/` |
| BMAD runtime config | BMAD module config, manifests, and 5 Python helper scripts (tooling, not app code) | `_bmad/` |
| BMAD output | Empty drop zone for BMAD-generated artifacts | `_bmad-output/` |
| GSD analysis output | Codebase maps written by `/gsd:map-codebase` subagents | `.planning/codebase/` |
| Legacy drafts | Historical Module 03 drafts, superseded by `docs/modules/03-forms-workflow-builder/` | `functional-spec_1.md`, `high-level-design.md`, `low-level-design_1.md`, `architecture-diagram_1.md` |

## Pattern Overview

**Overall:** Layered documentation-as-specification package ("spec monorepo") with a
shared-context/extend-don't-repeat rule, wrapped in a two-layer AI engineering workflow
(BMAD planning + GSD execution).

**Key Characteristics:**
- **Fixed document schema.** Every one of the 25 module folders contains exactly the same five
  files (`README.md`, `functional-spec.md`, `high-level-design.md`, `low-level-design.md`,
  `architecture-diagram.md`) — verified across all 25 folders. Depth expectations per file type are
  codified in `docs/_context/conventions.md` ("Document depth expectations per file").
- **Single source of truth per concern.** Cross-cutting material lives once in `docs/_context/`;
  module docs assume it and never repeat it (rule stated in `docs/_context/README.md` "The rules").
  `docs/_context/engineering-decisions.md` is explicitly BINDING: where a module doc is silent, the
  register's answer wins.
- **Contract-before-code.** Concurrency is prevented by convention, not compilation: canonical
  entity names, state machines, API error shapes, and event envelopes are fixed in
  `docs/_context/conventions.md` and `docs/01-FSD.md` §4–§6; module docs must extend, never
  rename or contradict.
- **Traceability spine.** Numbered requirement IDs (`FR-<MOD>-NNN`) trace to epic/story IDs
  (`EPIC-xx`, `EPIC-xx.S-yy`) in `docs/02-Epics-and-Stories.md` (rule in
  `docs/_context/conventions.md` "Traceability").
- **Human-readable entry, machine-consumable depth.** Plain-language `README.md` at repo root, in
  `docs/modules/README.md`, and in every module folder; deep-dive specs behind them.

## Layers

**Layer 1 — Index / Navigation:**
- Purpose: Route readers and AI agents to the right document; declare workflow rules.
- Location: `README.md`, `CLAUDE.md`, `docs/README.md`
- Contains: Navigation tables, reading order, AI routing rules (BMAD = planning only, GSD = all code execution).
- Depends on: Nothing.
- Used by: Every other layer; `CLAUDE.md` is auto-loaded context for all AI agents in this workspace.

**Layer 2 — Shared Context:**
- Purpose: The single definition of stack, entities, formats, and binding decisions that every module doc inherits.
- Location: `docs/_context/` (5 files)
- Contains: `source-spec.md` (baseline scope), `conventions.md` (house rules), `engineering-decisions.md` (binding register), `new-modules-addendum.md` (modules 14–25 scope), `README.md` (layer index + rules).
- Depends on: Nothing (this is the foundation).
- Used by: All 25 module folders, `docs/01-FSD.md`, `docs/02-Epics-and-Stories.md`.

**Layer 3 — Platform Specification:**
- Purpose: Product-wide and platform-wide build-ready specs plus the engineering backlog.
- Location: `docs/00-PRD.md`, `docs/01-FSD.md`, `docs/02-Epics-and-Stories.md`
- Contains: PRD v3.0; FSD v2.1 (§1 Architecture Overview, §2 ADR-001…009, §3 Core Technical Components, §4 Domain Model, §5 State Machines, §6 API & Event Platform, §7 Offline Sync, §8 AI Runtime & Governance, §9 Security, §10 Observability, §11 NFRs, §12 Cross-Module Contracts); 15 epics / 154 stories.
- Depends on: Layer 2 (`01-FSD.md` incorporates `engineering-decisions.md` by reference, §1.3).
- Used by: Layer 4 (every module HLD/LLD must build against it) and the backlog.

**Layer 4 — Module Specification:**
- Purpose: Per-module build-ready detail: requirements, service boundaries, schemas, API contracts, diagrams.
- Location: `docs/modules/01-identity-organization/` … `docs/modules/25-field-service-vendor-scheduling/` (25 folders, identical shape), catalog at `docs/modules/README.md`
- Contains: Fixed 5-file set per module. Core modules 01–13 map to the original baseline; gap modules 14–25 reuse core primitives (`WorkItem`, `Evidence`, `Issue`, `TemplateVersion`, shared `ApprovalRoute`) instead of building parallel engines (rule in `docs/modules/README.md` "How the modules fit together").
- Depends on: Layers 2 and 3 exclusively (module docs never contradict them).
- Used by: Implementers; future GSD phases via `.planning/codebase/` maps.

**Layer 5 — AI Workflow Tooling:**
- Purpose: The two-layer AI engineering system that plans (BMAD) and executes (GSD) work in this repo.
- Location: `.claude/` (commands, agents, get-shit-done framework v1.42.3, hooks, skills), `.zcode/skills/` (byte-identical copy of the 36 BMAD skills — verified with `diff`), `_bmad/` (BMAD config/manifests/scripts), `_bmad-output/` (empty output dir)
- Contains: 67 GSD slash commands in `.claude/commands/gsd/`; 33 `gsd-*` subagent definitions in `.claude/agents/`; ~80 workflow definitions + references + templates under `.claude/get-shit-done/`; 11 GSD hook scripts in `.claude/hooks/`; 36 `bmad-*` skills.
- Depends on: Layers 1–4 as source of truth (`CLAUDE.md` routes implementation questions to `docs/` and `.planning/codebase/`).
- Used by: AI sessions in this workspace.

## Data Flow

### Primary Reading Path (documentation "request path")

1. Reader/agent lands on `README.md` — the declared index ("this README is the index") — and picks a goal from the routing table.
2. Product context: `docs/00-PRD.md` (vision, personas, module map).
3. Binding decisions: `docs/_context/engineering-decisions.md` — required before any implementation work per `README.md` ("Implementing from these docs? Read in this order").
4. Platform architecture: `docs/01-FSD.md` (§1 architecture, §2 ADRs, §4 domain model, §5 state machines, §6 API/events).
5. Backlog: `docs/02-Epics-and-Stories.md` (epic/story selection).
6. Module deep dive: `docs/modules/<NN-name>/README.md` (plain language) → `functional-spec.md` → `high-level-design.md` → `low-level-design.md`, with `architecture-diagram.md` as visual companion (order defined in `docs/README.md` "How to read this package").

### Knowledge Extension Flow (how the package grows)

1. New cross-cutting decision/convention → added to the relevant `docs/_context/` file, never into module folders (rule in `docs/_context/README.md`: "New shared material belongs here").
2. New product capability → new numbered module folder under `docs/modules/` following the fixed 5-file schema; rationale recorded in `docs/_context/new-modules-addendum.md` (this is how modules 14–25 were added).
3. Catalogs updated: entry added to `docs/modules/README.md` and `docs/README.md`; repo-level summary updated in `README.md`.

### AI Execution Flow (per `CLAUDE.md` routing rules)

1. Vague idea / major new epic → BMAD planning personas only (`.claude/skills/bmad-*`); formal specs written into `docs/`; BMAD then stops — BMAD dev agents never write application code.
2. Implementation work → GSD loop only: `/gsd:discuss-phase` → `/gsd:plan-phase` → `/gsd:execute-phase` → `/gsd:verify-work` (commands in `.claude/commands/gsd/`), with `docs/` and `.planning/codebase/` treated as strict source of truth.

**State Management:**
- No runtime state exists. Workflow state lives in markdown: GSD planning artifacts belong under
  `.planning/` (this map is the first content there); BMAD outputs under `_bmad-output/` (currently
  empty). Git history is the only versioning mechanism (`.git/` present; `.gitignore` excludes
  `.DS_Store`, `*.zip`, `*.zip.tmp`).

## Key Abstractions

**The Module Quartet (doc-level abstraction):**
- Purpose: A fixed 4-document decomposition (functional requirements → service design → schema/API detail → diagrams) plus plain-language README, applied uniformly to all 25 modules.
- Examples: `docs/modules/01-identity-organization/functional-spec.md` … `architecture-diagram.md`; identical set in all 25 folders.
- Pattern: Template-driven uniformity; depth contract enforced by `docs/_context/conventions.md` ("Document depth expectations per file").

**Canonical Domain Entities (spec-level abstraction):**
- Purpose: Fixed vocabulary preventing entity drift across 25 module specs: `Tenant`, `OrganizationNode`, `User`, `UserScope`, `Role`, `Template`, `TemplateVersion`, `Submission`, `WorkItem`, `Issue`, `Evidence`, `KnowledgeDocument`, `AgentRun`, `AgentToolExecution`, `AuditLog`, `Asset`, `Course`/`LearningAssignment`, `Integration`, `WebhookSubscription`, `NotificationPreference`.
- Examples: Field-level detail in `docs/01-FSD.md` §4.2; entity naming rule in `docs/_context/conventions.md` ("Domain Model — canonical entities").
- Pattern: Extend-don't-rename; gap modules (14–25) map their entities to these primitives via `docs/_context/new-modules-addendum.md`.

**Fixed Lifecycle State Machines (spec-level abstraction):**
- Purpose: Reusable, closed state machines: `WorkItem` (DRAFT→…→VERIFIED with BLOCKED/REJECTED/OVERDUE/ESCALATED branches), `Issue` (OPEN→…→CLOSED with REOPENED), `TemplateVersion` (DRAFT→…→RETIRED), agent autonomy levels L0–L4.
- Examples: `docs/01-FSD.md` §5.1–§5.4; verbatim copies required by `docs/_context/conventions.md` ("State machines").
- Pattern: "Reuse verbatim; do not invent new terminal states."

**Requirement & Story IDs (traceability abstraction):**
- Purpose: Cross-document referencing without duplication: `FR-<MOD>-NNN` functional requirements, `EPIC-xx` epics, `EPIC-xx.S-yy` stories.
- Examples: `docs/modules/01-identity-organization/functional-spec.md` uses `FR-` IDs; `docs/02-Epics-and-Stories.md` owns `EPIC-xx.S-yy`.
- Pattern: Numbered, stable identifiers; mapping table (note: epic numbers intentionally do not align 1:1 with folder numbers — stated in `docs/modules/README.md` "Related indexes").

**ADR Records (decision abstraction):**
- Purpose: Immutable recorded architecture decisions with context/rationale.
- Examples: ADR-001 (modular monolith at domain-service boundary), ADR-002 (Postgres RLS mandatory), ADR-003 (Kafka single backbone), ADR-004 (Temporal), ADR-005 (WatermelonDB), ADR-006 (OPA/Rego), ADR-007 (pgvector default), ADR-008 (Gateway/BFF per-client shaping), ADR-009 (TimescaleDB for IoT) — all in `docs/01-FSD.md` §2.
- Pattern: Classic ADR format; binding unless a new ADR supersedes.

## Entry Points

**No runtime code entry points exist.** There is no `main`, no server bootstrap, no deployable
artifact anywhere in the repository. The entry points below are **documentation entry points**.

**Human entry point:**
- Location: `README.md`
- Triggers: Anyone opening the repo.
- Responsibilities: Declares spec-only nature; three-line product summary; routing table to PRD/FSD/backlog/catalog/context; module grouping summary; canonical reading order; flags root `_1`-suffixed files as legacy drafts of Module 03.

**AI agent entry point:**
- Location: `CLAUDE.md`
- Triggers: Loaded automatically as context for every AI session in this workspace.
- Responsibilities: Declares the repo spec-only; mandates BMAD-vs-GSD routing; names `docs/`, `.planning/codebase/` as strict source of truth; sets quality guardrails (tests/lint/typecheck before task completion).

**Documentation index entry points:**
- Location: `docs/README.md` — file-by-file index of the whole package with reading order.
- Location: `docs/modules/README.md` — module catalog (one-line per module, all 25) plus the cross-module integration patterns ("How the modules fit together").
- Location: `docs/_context/README.md` — index of the shared reference layer and its binding rules.

**Tool entry points:**
- Location: `.claude/commands/gsd/` — 67 GSD slash commands (`/gsd:map-codebase`, `/gsd:plan-phase`, `/gsd:execute-phase`, …).
- Location: `_bmad/config.toml`, `_bmad/_config/manifest.yaml` — BMAD runtime configuration consumed by BMAD skills.

## Architectural Constraints

- **No-code constraint:** The repo must stay spec-only; all implementation belongs in future
  polyrepo services per `docs/_context/engineering-decisions.md` §4 (`platform-contracts`,
  `svc-*`, `mobile-app`, `web-admin`, `infra`) — not in this repository.
- **Binding documents:** `docs/_context/engineering-decisions.md` and `docs/_context/conventions.md`
  override any module document. A module doc may be extended, never contradicted
  (`docs/_context/README.md`, `docs/01-FSD.md` §1.3).
- **No alternative stacks:** `docs/_context/conventions.md` "Target Tech Stack" is fixed
  ("do not invent alternatives"); `README.md` "Repo notes" repeats this as a hard rule.
- **Format constraints:** All diagrams must be Mermaid code blocks (`docs/_context/conventions.md`
  "Diagram conventions"); requirement IDs must be numbered and traceable; no "TBD" placeholders —
  make an explicit decision or mark `Decision needed:` with a recommended default
  (`docs/_context/conventions.md` intro).
- **Uniform module shape:** Any new module folder must contain exactly `README.md`,
  `functional-spec.md`, `high-level-design.md`, `low-level-design.md`, `architecture-diagram.md`.
- **Tooling duplication constraint:** `.claude/skills/` and `.zcode/skills/` are maintained as
  identical trees (verified byte-identical via `diff -rq`); BMAD skill changes must be applied to
  both. GSD framework files under `.claude/get-shit-done/` carry their own version
  (`.claude/get-shit-done/VERSION` = 1.42.3) and update hooks (`.claude/hooks/gsd-check-update.js`).
- **Legacy file quarantine:** Root `_1`-suffixed drafts (`functional-spec_1.md`,
  `low-level-design_1.md`, `architecture-diagram_1.md`) and `high-level-design.md` are historical
  copies of Module 03 content; the canonical versions live in
  `docs/modules/03-forms-workflow-builder/` — never edit the root drafts to change behavior.
- **No circular references by design:** the dependency direction is strictly downward
  (modules → platform docs → `_context`); `01-FSD.md` deliberately incorporates
  `engineering-decisions.md` by reference instead of duplicating it, to avoid drift
  (`docs/01-FSD.md` §1.3).

## Anti-Patterns

### Duplicating shared context inside module documents

**What happens:** Copying stack rules, entity definitions, or state machines from
`docs/_context/` into a module spec.
**Why it's wrong:** The package's drift-prevention model depends on single-source definitions;
copies diverge silently across 25 modules. The rule is explicit in `docs/_context/README.md`
("Module documents assume this folder and never repeat it").
**Do this instead:** Reference the `_context` file by name; put only module-specific content in
`docs/modules/<NN-name>/`. Follow the pattern of `docs/01-FSD.md` §1.3, which incorporates the
register by reference.

### Building parallel engines in gap modules

**What happens:** A new module (14–25) defines its own approval engine, scheduler, or evidence store.
**Why it's wrong:** The platform deliberately reuses Work Orchestration's `ApprovalRoute`, the one
WorkItem lifecycle, and the immutable Evidence model; parallel engines break the "one mechanism"
guarantees documented in `docs/modules/README.md` "How the modules fit together".
**Do this instead:** Express the module's needs as reuse of core primitives, as done in
`docs/modules/19-inventory-procurement/` (purchase approvals route through Work Orchestration) and
`docs/modules/22-gamification/` (points derive from already-verified work).

### Editing legacy root drafts instead of canonical module docs

**What happens:** Changing Module 03 behavior by editing `functional-spec_1.md` or
`low-level-design_1.md` at the repo root.
**Why it's wrong:** `README.md` "Repo notes" declares these files legacy history; the canonical
Module 03 docs live in `docs/modules/03-forms-workflow-builder/`. Dual editing guarantees divergence.
**Do this instead:** Always edit `docs/modules/03-forms-workflow-builder/*.md`; leave root drafts untouched.

### Mutating captured evidence (specified anti-pattern)

**What happens:** (In the target system) overwriting or editing an `Evidence` record after capture.
**Why it's wrong:** Evidence is hash-protected with chain of custody; mutation destroys the
tamper-evidence the compliance story depends on (rule in `docs/modules/README.md`: "Evidence (05)
is never mutated").
**Do this instead:** Model corrections as new linked records (AI verdicts, redactions), per
`docs/modules/05-evidence-management/`.

### Trusting client-asserted context (specified anti-pattern)

**What happens:** (In the target system) honoring a client-supplied `tenant_id` header or client clock.
**Why it's wrong:** Tenant scope and server authority are the platform's isolation backbone
(`docs/_context/conventions.md` "Multi-tenancy & Scoping Rule"; `docs/modules/README.md`:
"Server is the source of truth").
**Do this instead:** Resolve tenant from the authenticated principal at the gateway; key offline
writes on client-generated UUIDs with idempotent sync (`docs/01-FSD.md` §7).

## Error Handling

**Strategy:** Not applicable at runtime (no code). The package's analog is its **ambiguity
handling convention**: documents must never contain "TBD" — authors make an explicit engineering
decision, or mark it `Decision needed:` with a recommended default so work is not blocked
(`docs/_context/conventions.md`, intro section). Additionally, `docs/_context/engineering-decisions.md`
states that every module's former "Open Questions/Assumptions" section was rewritten as
"Resolved Decisions" — the package is designed to contain no build-blocking ambiguity.

**Patterns:**
- Silence resolution: where a module doc is silent on a covered topic, the binding register's
  answer applies (`docs/_context/engineering-decisions.md`).
- Conflict resolution between docs: precedence order is `engineering-decisions.md` /
  `conventions.md` > `01-FSD.md` > module docs (`docs/01-FSD.md` preamble; `CLAUDE.md` declares
  these the strict source of truth).
- Target-system error contract is specified content: the uniform error envelope
  `{ "error": { "code", "message", "trace_id" } }` with stable enumerated codes
  (`docs/_context/conventions.md` "API conventions").

## Cross-Cutting Concerns

**Logging:** N/A at runtime. The target observability model (OpenTelemetry traces/metrics/logs
correlated by `trace_id`) is specified in `docs/01-FSD.md` §10 and
`docs/_context/engineering-decisions.md` §9.

**Validation:** Consistency review against `docs/_context/conventions.md` is the package's
validation gate — every doc must use the same stack, entity names, API/event formats, state
machines, and Mermaid style (`docs/_context/README.md` "The rules"). Traceability of every `FR-`
requirement to an `EPIC-xx` story is a stated requirement (`docs/_context/conventions.md`
"Traceability").

**Authentication:** N/A at runtime. Target auth mechanics (OAuth 2.1/OIDC, DPoP-bound
sender-constrained tokens, refresh-token rotation, 72-hour offline grace, mTLS + internal JWTs for
service-to-service) are binding spec content in `docs/_context/engineering-decisions.md` §1,
summarized in `docs/01-FSD.md` §1.3.

**Documentation-as-source-of-truth for AI:** `CLAUDE.md` elevates `docs/architecture.md`,
`docs/PRD.md`, and `.planning/codebase/` to "strict source of truth" for all AI-driven execution;
subagents like `.claude/agents/gsd-codebase-mapper.md` write analysis artifacts here to keep
orchestrator context small.

---

*Architecture analysis: 2026-11-27*
