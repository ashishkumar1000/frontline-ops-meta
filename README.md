# Frontline Operations & Agentic Ops Platform — Specification Package

A build-ready specification repository for a **mobile-first, work-class B2B SaaS platform** that runs
frontline operations across distributed physical locations — retail chains, QSR and hospitality
groups, facilities and logistics networks, manufacturing floors, and healthcare-adjacent site
operations.

The platform closes the loop from **standard → work → evidence → exception → correction →
re-verification → learning** in one governed system, replacing the fragmented stack of checklist
tools, standalone LMSs, chat apps, spreadsheet trackers, and retrofitted BI dashboards most
enterprises patch together today.

> **This repo contains specifications, not source code.** Everything here is written at build-ready
> depth — concrete schemas, endpoint contracts, state machines, and edge cases — so engineering teams
> (or coding agents) can implement directly from these documents.

## What's in the package

| Metric | Count |
|---|---|
| Product modules specified | 25 (13 core + 12 competitive-gap extensions) |
| Epics decomposed | 15 |
| Build-ready user stories | 154 (with acceptance criteria, priority, sizing) |
| Documents per module | 4 (functional spec, HLD, LLD, architecture diagrams) |

## Repository layout

```
.
├── README.md                          ← you are here
├── frontline-ops-spec-package.zip     ← archive of the original distributed package
├── docs/
│   ├── 00-PRD.md                      Master PRD v3.0 (vision, personas, roadmap, metrics)
│   ├── 01-FSD.md                      Platform-wide functional spec (architecture, ADRs, domain model)
│   ├── 02-Epics-and-Stories.md        The engineering backlog: 15 epics → 154 stories
│   ├── _context/                      Shared reference material (read before any module doc)
│   │   ├── source-spec.md             Original v2.0 PRD/FSD baseline (source of truth for scope)
│   │   ├── conventions.md             Tech stack, domain model, API/event/diagram conventions
│   │   ├── engineering-decisions.md   BINDING cross-cutting decisions register
│   │   └── new-modules-addendum.md    Rationale + integration rules for modules 14–25
│   └── modules/                       25 module folders, each with:
│       └── NN-<slug>/                   functional-spec.md · high-level-design.md
│                                       low-level-design.md · architecture-diagram.md
├── architecture-diagram_1.md          ⚠️ Legacy draft (Module 03) — superseded by docs/modules/03/
├── functional-spec_1.md               ⚠️ Legacy draft (Module 03)
├── high-level-design.md               ⚠️ Legacy draft (Module 03)
└── low-level-design_1.md              ⚠️ Legacy draft (Module 03)
```

## The platform in one page

**The problem.** Standards, brand promises, safety rules, and compliance obligations are defined
centrally, but the work that fulfils them happens at the edge — on a phone or a clipboard, often
offline, invisible to the people accountable for it. The gap between *what leadership believes is
happening* and *what is actually happening at site level* is discovered only after a customer
complaint, a health inspection, an audit finding, or a safety incident.

**The bet.** Mobile-first, explicitly. The frontline mobile app is the primary product surface for
the majority of users (associates, auditors, most site managers); the web Control Tower serves the
administrative and analytical minority. Offline is a first-class operating mode, not a fallback.

**The moat.** The **Operational Graph** — the linked record of organization, work, evidence,
exceptions, knowledge, and historical interventions that makes an AI agent's recommendation
trustworthy and an auditor's evidence defensible. AI capabilities (visual compliance checking,
agentic operations, NL workflow authoring) are governed by deterministic policy — ungoverned AI in a
regulated, multi-tenant, safety-relevant domain is a liability, not a feature.

## Modules

### Core modules (01–13, from the v2.0 baseline)

| # | Module | What it owns |
|---|---|---|
| 01 | [Identity & Organization](docs/modules/01-identity-organization/) | Tenants, arbitrary-depth org trees, RBAC + scoping |
| 02 | [Work Orchestration](docs/modules/02-work-orchestration/) | WorkItems, scheduling, SLA timers, escalation, approvals |
| 03 | [Forms & Workflow Builder](docs/modules/03-forms-workflow-builder/) | No-code versioned templates, logic rules, submissions |
| 04 | [Frontline Mobile & Offline](docs/modules/04-mobile-offline/) | Offline-first sync, media capture, background work |
| 05 | [Evidence Management](docs/modules/05-evidence-management/) | Tamper-evident media, signed URLs, retention |
| 06 | [Issues & CAPA](docs/modules/06-issues-capa/) | Exception lifecycle, corrective/preventive actions |
| 07 | [Communications & Knowledge](docs/modules/07-communications-knowledge/) | Targeted comms, knowledge hub, RAG retrieval |
| 08 | [Learning](docs/modules/08-learning/) | Courses, assignments, competency tracking |
| 09 | [Visual AI](docs/modules/09-visual-ai/) | Governed computer-vision compliance checking |
| 10 | [Agentic Operations](docs/modules/10-agentic-operations/) | Agent runtime, typed tool-calling, OPA policy |
| 11 | [Analytics & Control Tower](docs/modules/11-analytics-control-tower/) | Cross-module reporting, warehouse, BI |
| 12 | [Integrations & Extensibility](docs/modules/12-integrations-extensibility/) | Webhooks, APIs, third-party connectors |
| 13 | [Security, Privacy & Compliance](docs/modules/13-security-compliance/) | Audit, DPoP, EU AI Act posture, data protection |

### Competitive-gap modules (14–25)

Added after a gap analysis against Wooqer, Zenput, SafetyCulture, YOOBIC, Connecteam, MaintainX, and
Operandio ([rationale](docs/_context/new-modules-addendum.md)). They extend the platform using the
same core primitives (`WorkItem`, `Evidence`, `Issue`, `TemplateVersion`, multi-level approval
routing) rather than duplicating them.

| Tier | Modules |
|---|---|
| 1 | [14 Workforce Scheduling](docs/modules/14-workforce-scheduling/) · [15 Time & Attendance](docs/modules/15-time-attendance/) · [16 Asset Maintenance (CMMS)](docs/modules/16-asset-maintenance-cmms/) |
| 2 | [17 IoT Sensor Platform](docs/modules/17-iot-sensor-platform/) · [18 Employee Engagement](docs/modules/18-employee-engagement/) · [19 Inventory & Procurement](docs/modules/19-inventory-procurement/) · [20 EHS — Safety & Risk](docs/modules/20-ehs-safety-risk/) |
| 3 | [21 Food Safety & HACCP](docs/modules/21-food-safety-haccp/) · [22 Gamification](docs/modules/22-gamification/) · [23 Store Launch Management](docs/modules/23-store-launch-management/) · [24 App Marketplace](docs/modules/24-app-marketplace/) · [25 Field Service & Vendor Scheduling](docs/modules/25-field-service-vendor-scheduling/) |

## Target tech stack

Defined once in [`_context/conventions.md`](docs/_context/conventions.md) and used consistently
across every module — do not invent alternatives when implementing:

| Layer | Choice |
|---|---|
| Mobile app | React Native (TypeScript), WatermelonDB, native modules for camera/sync/biometrics |
| Web Control Tower | React + TypeScript behind a BFF aggregation layer |
| Backend services | Kotlin + Spring Boot microservices, one deployable per bounded context |
| Datastore | PostgreSQL with `tenant_id` + RLS row-level multi-tenancy |
| Event backbone | Kafka, tenant-partitioned by `tenant_id:aggregate_id` |
| Workflow engine | Temporal.io (schedules, SLA timers, escalation ladders, CAPA state machines) |
| Search / RAG | OpenSearch + pgvector behind the Knowledge Hub and Agent Runtime |
| Object storage | S3-compatible, per-tenant prefix, KMS-encrypted signed URLs |
| Auth | OAuth 2.1 + OIDC, DPoP sender-constrained tokens (see [decisions register](docs/_context/engineering-decisions.md)) |
| AI gateway | Provider-agnostic (LiteLLM-style) fronting OpenAI/Anthropic/Azure/Bedrock; OPA/Rego policy |
| Deploy / IaC | Kubernetes, Terraform, Helm, Argo Rollouts progressive delivery |
| Observability | OpenTelemetry traces/metrics/logs, correlated by `trace_id` |

## How to read this package

1. **[00-PRD.md](docs/00-PRD.md)** — product context, personas, mobile-first strategy, module map.
2. **[_context/engineering-decisions.md](docs/_context/engineering-decisions.md)** — **read before
   any implementation work.** Binding across every module: auth mechanics, versioning, environments,
   testing gates, coding standards, Definition of Done. Module docs assume it and do not repeat it.
3. **[01-FSD.md](docs/01-FSD.md)** — the platform architecture every module builds on.
4. **[02-Epics-and-Stories.md](docs/02-Epics-and-Stories.md)** — the engineering backlog.
5. **`docs/modules/<name>/`** — build-ready detail per module, in order:
   `functional-spec.md` → `high-level-design.md` → `low-level-design.md`, with
   `architecture-diagram.md` as the quick-reference visual companion.

All diagrams are Mermaid code blocks and render natively on GitHub, GitLab, VS Code (Mermaid
extension), Notion, and most modern markdown viewers.

## Conventions for contributors

- New/edited documents **must** stay consistent with
  [`_context/conventions.md`](docs/_context/conventions.md) (entity names, API conventions,
  tech stack) and **must not contradict**
  [`_context/engineering-decisions.md`](docs/_context/engineering-decisions.md).
- Use canonical entity names exactly as defined in the FSD: `Tenant`, `OrganizationNode`, `User`,
  `UserScope`, `Role`, `Template`, `TemplateVersion`, `Submission`, `WorkItem`, `Issue`, `Evidence`,
  `KnowledgeDocument`, `AgentRun`, `AgentToolExecution`, `AuditLog`, `Asset`, `Course`,
  `LearningAssignment`, `Integration`, `WebhookSubscription`, `NotificationPreference`.
- No vague placeholders. Make an explicit engineering decision and state it; where genuinely open,
  mark `Decision needed:` with a recommended default so work is never blocked.
- Module folders carry four files only — keep new shared material in `_context/`.
