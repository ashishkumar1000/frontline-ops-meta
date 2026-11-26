# Frontline Operations & Agentic Ops Platform — Build-Ready Specification Package

This package supersedes and deepens the original v2.0 PRD/FSD baseline (`_context/source-spec.md`)
into a build-ready set of documents: an enhanced overall PRD and FSD, an epic/story backlog, and a
full Functional Spec + High-Level Design + Low-Level Design + Architecture Diagram set for each of
the 13 product modules.

Shared conventions (tech stack, domain model, API/event conventions, diagram format) that every
document below is written against live in `_context/conventions.md`. Read that first if you are
picking up any single module doc in isolation.

## Top-level documents

| File | Contents |
|---|---|
| [`00-PRD.md`](./00-PRD.md) | Enhanced Product Requirements Document — vision, personas, mobile-first strategy, principles, capability map, module overviews, roadmap, risks, success metrics, glossary |
| [`01-FSD.md`](./01-FSD.md) | Enhanced platform-wide Functional Specification Document — architecture, ADRs, domain model, API/event platform, offline sync, AI governance, security, observability, NFRs, cross-module contracts |
| [`02-Epics-and-Stories.md`](./02-Epics-and-Stories.md) | All 15 epics decomposed into ~150 build-ready user stories with acceptance criteria, priority and sizing |

## Module documentation

Each module folder under `modules/` contains a plain-language `README.md` alongside four deep-dive docs:
- `README.md` — a plain-language overview of the module: what it does, who uses it, and how it fits the platform, with pointers into the four deep-dive docs
- `functional-spec.md` — numbered functional requirements, UX states, edge cases, NFRs, dependencies
- `high-level-design.md` — service boundaries, component diagram, data ownership, scalability, tech choices
- `low-level-design.md` — DB schema, API contracts, sequence diagrams, algorithms, error handling
- `architecture-diagram.md` — focused Mermaid diagrams (component, sequence, ER, state)

| # | Module | Epics | Docs |
|---|---|---|---|
| 01 | Identity & Organization | EPIC-01, EPIC-02 | [`README`](./modules/01-identity-organization/README.md) · [`modules/01-identity-organization/`](./modules/01-identity-organization/) |
| 02 | Work Orchestration | EPIC-03 | [`README`](./modules/02-work-orchestration/README.md) · [`modules/02-work-orchestration/`](./modules/02-work-orchestration/) |
| 03 | Forms & Workflow Builder | EPIC-04 | [`README`](./modules/03-forms-workflow-builder/README.md) · [`modules/03-forms-workflow-builder/`](./modules/03-forms-workflow-builder/) |
| 04 | Frontline Mobile & Offline | EPIC-05 | [`README`](./modules/04-mobile-offline/README.md) · [`modules/04-mobile-offline/`](./modules/04-mobile-offline/) |
| 05 | Evidence Management | EPIC-06 | [`README`](./modules/05-evidence-management/README.md) · [`modules/05-evidence-management/`](./modules/05-evidence-management/) |
| 06 | Issues & CAPA | EPIC-07 | [`README`](./modules/06-issues-capa/README.md) · [`modules/06-issues-capa/`](./modules/06-issues-capa/) |
| 07 | Communications & Knowledge | EPIC-08 | [`README`](./modules/07-communications-knowledge/README.md) · [`modules/07-communications-knowledge/`](./modules/07-communications-knowledge/) |
| 08 | Learning | EPIC-09 | [`README`](./modules/08-learning/README.md) · [`modules/08-learning/`](./modules/08-learning/) |
| 09 | Visual AI | EPIC-10 | [`README`](./modules/09-visual-ai/README.md) · [`modules/09-visual-ai/`](./modules/09-visual-ai/) |
| 10 | Agentic Operations (Agent Runtime) | EPIC-11 | [`README`](./modules/10-agentic-operations/README.md) · [`modules/10-agentic-operations/`](./modules/10-agentic-operations/) |
| 11 | Analytics & Control Tower | EPIC-12, EPIC-15 | [`README`](./modules/11-analytics-control-tower/README.md) · [`modules/11-analytics-control-tower/`](./modules/11-analytics-control-tower/) |
| 12 | Integrations & Extensibility | EPIC-13 | [`README`](./modules/12-integrations-extensibility/README.md) · [`modules/12-integrations-extensibility/`](./modules/12-integrations-extensibility/) |
| 13 | Security, Privacy & Compliance | EPIC-14 | [`README`](./modules/13-security-compliance/README.md) · [`modules/13-security-compliance/`](./modules/13-security-compliance/) |

### Competitive-gap modules (14–25)

Added after a gap analysis against Wooqer, Zenput, SafetyCulture, YOOBIC, Connecteam, MaintainX and
Operandio — see `_context/new-modules-addendum.md` for the full rationale and integration map. These
are not in the original v2.0 baseline; they extend the platform using the same primitives (WorkItem,
Evidence, Issue, TemplateVersion, the multi-level Approval Routing built for Work Orchestration,
etc.) rather than duplicating them.

| # | Module | Tier | Docs |
|---|---|---|---|
| 14 | Workforce Scheduling & Labor Management | 1 | [`README`](./modules/14-workforce-scheduling/README.md) · [`modules/14-workforce-scheduling/`](./modules/14-workforce-scheduling/) |
| 15 | Time & Attendance | 1 | [`README`](./modules/15-time-attendance/README.md) · [`modules/15-time-attendance/`](./modules/15-time-attendance/) |
| 16 | Asset & Equipment Maintenance (CMMS) | 1 | [`README`](./modules/16-asset-maintenance-cmms/README.md) · [`modules/16-asset-maintenance-cmms/`](./modules/16-asset-maintenance-cmms/) |
| 17 | IoT Sensor Platform | 2 | [`README`](./modules/17-iot-sensor-platform/README.md) · [`modules/17-iot-sensor-platform/`](./modules/17-iot-sensor-platform/) |
| 18 | Employee Engagement & Voice of Frontline | 2 | [`README`](./modules/18-employee-engagement/README.md) · [`modules/18-employee-engagement/`](./modules/18-employee-engagement/) |
| 19 | Inventory, Procurement & Supplier Management | 2 | [`README`](./modules/19-inventory-procurement/README.md) · [`modules/19-inventory-procurement/`](./modules/19-inventory-procurement/) |
| 20 | EHS — Safety Incident & Risk Register | 2 | [`README`](./modules/20-ehs-safety-risk/README.md) · [`modules/20-ehs-safety-risk/`](./modules/20-ehs-safety-risk/) |
| 21 | Food Safety & HACCP | 3 | [`README`](./modules/21-food-safety-haccp/README.md) · [`modules/21-food-safety-haccp/`](./modules/21-food-safety-haccp/) |
| 22 | Gamification & Recognition | 3 | [`README`](./modules/22-gamification/README.md) · [`modules/22-gamification/`](./modules/22-gamification/) |
| 23 | New-Location Launch Management | 3 | [`README`](./modules/23-store-launch-management/README.md) · [`modules/23-store-launch-management/`](./modules/23-store-launch-management/) |
| 24 | App Marketplace & Template Catalog | 3 | [`README`](./modules/24-app-marketplace/README.md) · [`modules/24-app-marketplace/`](./modules/24-app-marketplace/) |
| 25 | Field Service & Third-Party Vendor Scheduling | 3 | [`README`](./modules/25-field-service-vendor-scheduling/README.md) · [`modules/25-field-service-vendor-scheduling/`](./modules/25-field-service-vendor-scheduling/) |

## Reference material

- [`_context/README.md`](./_context/README.md) — start here for the shared reference layer: what each `_context` file is for and the rules every document in this package follows
- `_context/source-spec.md` — extracted text of the original v2.0 PRD/FSD baseline document (source of truth for scope/terminology, modules 01–13)
- `_context/conventions.md` — shared tech stack, domain model, API/event/diagram conventions used across every document in this package
- `_context/new-modules-addendum.md` — scope, entity map, and integration rules for the 12 competitive-gap modules (14–25)
- **`_context/engineering-decisions.md`** — the binding, platform-wide Engineering Decisions Register: auth/OIDC mechanics, mobile OS support matrix, API versioning/deprecation, repo structure, CI/CD, environments, testing gates, secrets/observability/coding standards, Definition of Done, and the EU AI Act high-risk-AI compliance posture. **Read this before writing any code** — every module document assumes it and does not repeat it. Every module's former "Open Questions/Assumptions" section has been rewritten as "Resolved Decisions": there is no remaining build-blocking ambiguity in this package.

## How to read this package

1. Start with `00-PRD.md` for product context and the module map.
2. Read `_context/engineering-decisions.md` before any implementation work — it is binding across every module.
3. Read `01-FSD.md` for the platform architecture every module builds on.
4. Use `02-Epics-and-Stories.md` as the engineering backlog.
5. Go into each `modules/<name>/` folder for build-ready detail on that module, starting with
   `functional-spec.md`, then `high-level-design.md`, then `low-level-design.md`, with
   `architecture-diagram.md` as the quick-reference visual companion.

All diagrams are Mermaid code blocks and render natively in GitHub, GitLab, VS Code (with a Mermaid
extension), Notion, and most modern markdown viewers.
