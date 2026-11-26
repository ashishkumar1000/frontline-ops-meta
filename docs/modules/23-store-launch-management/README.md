# Module 23 — New-Location Launch Management

> Opening a new store, restaurant, or site is a multi-week project with dozens of dependent steps
> across construction, permitting, staffing, inventory, marketing, and grand opening — usually
> tracked in spreadsheets and email. This module turns that into a governed, repeatable project:
> an admin authors a reusable launch template, a regional manager instantiates it against a
> target location, and every checklist item becomes ordinary tracked work with a countdown,
> phase progress, and at-risk flags. Operations admins, regional/district managers, and store
> manager-designates are the primary users.

| | |
|---|---|
| **Module code** | LAUNCH (`FR-LAUNCH-###`) |
| **Owning service** | Launch Orchestration Service |
| **Tier** | 3 |

## In plain terms
- Build a launch template once: ordered phases (construction, permitting, staffing, inventory &
  merchandising, marketing, grand opening — plus custom ones), milestones like "Lease Signed" or
  "Opening Day," and the leaf checklist tasks under them.
- Wire dependencies between steps, e.g. "staffing cannot start until the Permits Approved
  milestone is verified" — validated acyclic at save time, translated 1:1 into WorkItem
  dependencies at launch.
- Turn a template into a real project bound to a location — either a brand-new location (the
  system creates its organization node automatically at a chosen trigger milestone, e.g. after
  "Lease Signed" verifies) or an existing one being remodeled or relaunched.
- See progress in launch language: phase completion percentage, days-to-launch countdown, and
  amber at-risk badges when a milestone is behind its expected schedule or its completion
  velocity projects a miss.
- Run many openings at once: a Gantt-style portfolio view across dozens of concurrent launches,
  filterable by status, template, region, and target open date.

## Key concepts
| Concept | What it means |
|---|---|
| `LaunchTemplate` / `LaunchTemplateVersion` | The reusable definition of what a launch looks like — immutable once published, versioned like any platform template. |
| `LaunchMilestone` | A named checkpoint in a phase, with an expected date offset from the target open date (or project start). |
| `LaunchProject` | One real launch: a template version bound to a target location, with status `PLANNED → IN_PROGRESS → AT_RISK → COMPLETE` (plus `CANCELLED`). |
| `LAUNCH_TASK` WorkItem | Every leaf checklist item, executed as a normal WorkItem in Work Orchestration (type `LAUNCH_TASK`, grouped under the project as a Campaign). |
| At-risk flag | A milestone behind its expected date — or projected to miss it at current completion velocity. |
| Days-to-launch | The countdown from today to the target open date, shown alongside the current phase everywhere. |

## How it connects to the rest
- **Work Orchestration (module 02 / EPIC-03)** — the module's defining relationship: a launch is
  a structured Campaign of dependent WorkItems. This module is explicitly a configuration/
  orchestration layer on top of Work Orchestration, not a new execution engine — it never
  re-implements assignment, the state machine, offline sync, or SLA/escalation; it bulk-creates
  `LAUNCH_TASK` WorkItems and consumes their events to roll up progress.
- **Identity & Organization (module 01 / EPIC-01/02)** — for new locations, this module calls the
  org node-creation API at the configured trigger milestone; node mechanics and scope propagation
  stay in Identity/Org.
- **Forms & Workflow Builder (module 03)** — a launch task can reference a `TemplateVersion` when
  execution needs a structured form or evidence (e.g. a final health inspection form).
- **Integrations & Extensibility (module 13 / EPIC-13)** — completing a task can fire an outbound
  feed (e.g. POS provisioning) via a configured integration trigger code.
- **Analytics & Control Tower (module 11 / EPIC-12/15)** — executive launch-pipeline rollups are
  computed in Analytics from this module's events and CDC feed; the module owns only its own
  portfolio dashboard.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-LAUNCH-001–057 covering template authoring, project instantiation, progress/at-risk tracking, the portfolio view, and template versioning. |
| [high-level-design.md](./high-level-design.md) | The thin "own the plan and the rollup, never the execution record" boundary, component diagram, and data ownership split with Work Orchestration. |
| [low-level-design.md](./low-level-design.md) | Schema and ER diagram, REST endpoints (template, instantiate, portfolio, progress), sequence diagrams, and pseudocode for template expansion and at-risk detection. |
| [architecture-diagram.md](./architecture-diagram.md) | Component diagram, instantiation and rollup sequence diagrams, owned-entity ER diagram, and the LaunchProject lifecycle state machine. |
