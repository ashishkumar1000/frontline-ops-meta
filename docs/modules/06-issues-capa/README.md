# Module 06 — Issues & CAPA

> When something goes wrong at a site — a fridge reads too warm, an audit check fails, safety gear is missing — this module turns it into a tracked issue: triage it, find the root cause, fix it, verify the fix worked, then close it. It is the platform's system of record for every managed deviation, with the full audit trail regulated industries expect. Store/site managers do most of the day-to-day work; operations admins set the rules; AI agents can also create and escalate issues.

| | |
|---|---|
| **Module code** | `06-issues-capa` |
| **Owning service** | Issue/CAPA Service |
| **Epics** | EPIC-07 (Issues & CAPA) |

## In plain terms
- Report a problem — or let the system create one from a failed form check, an overdue task, or an AI agent — and it becomes an **Issue** with a severity, an owner, and deadlines.
- Work the fix in steps: immediate correction, containment when it is serious (e.g., pull every unit from a suspect lot), then corrective and preventive actions — each step can require photo evidence.
- Record why it happened using a structured root cause method (5 Whys, Fishbone, Pareto, FMEA, or a tenant-defined custom one).
- Prove the fix worked — a re-inspection, document review, metric threshold, or manager attestation — before the issue can be resolved; a failed verification reopens it automatically.
- Nothing stalls quietly: per-severity SLA timers escalate step by step (notify the manager, raise the severity, executive alert) when deadlines slip.
- Spot repeat offenders: recurring failures feed a repeat-failure signal that Learning (module 08) and AI agents use to trigger targeted retraining.

## Key concepts
| Concept | What it means |
|---|---|
| Issue | A tracked deviation from an operating standard — e.g. "walk-in cooler at 10.2°C" — with a severity, an owner, and a lifecycle state. |
| Issue state machine | The fixed lifecycle `OPEN → TRIAGED → ASSIGNED → INVESTIGATING → CORRECTIVE_ACTION → VERIFICATION → RESOLVED → CLOSED` (plus a `REOPENED` branch); every other module reuses these exact state names. |
| CAPA step | One record in the fix workflow (immediate correction, containment, corrective action, preventive action). CAPA stands for Corrective and Preventive Action. |
| Root cause analysis | The structured "why did this happen" record; it is versioned, so switching method mid-investigation preserves the earlier attempt. |
| Effectiveness verification | Proof the fix worked, required before `RESOLVED`; for HIGH/CRITICAL issues the verifier must be someone other than the issue owner. |
| Severity | `LOW` / `MEDIUM` / `HIGH` / `CRITICAL`; drives SLA targets and which steps are mandatory (HIGH/CRITICAL require containment). |
| SLA & escalation ladder | Per-severity time targets (e.g. CRITICAL triaged within 15 minutes) with escalating actions, driven by one durable timer workflow per issue. |
| Recurrence link | A `CLOSED` issue is immutable and can never be reopened; a genuine repeat creates a **new** Issue linked via `recurrence_of_issue_id`. |

## How it connects to the rest
- Depends on **Work Orchestration (02)**: corrective actions and re-inspections are real WorkItems on the frontline Today screen, and an overdue task can escalate into an Issue here.
- Depends on **Evidence (05)** for photo/video proof at every CAPA step, and on the **Form Platform (03)**, whose rule engine creates issues when a form submission breaks a tenant rule.
- Feeds **Learning (08)**: repeat-failure patterns trigger targeted retraining, and if training does not fix the failure, Learning escalates the case back here.
- Exposes typed agent tools (`create_issue`, `get_recent_failures`, `schedule_reinspection`) to **Agentic Ops (10)**, and emits `issue.*` events consumed by **Analytics (11)** for KPIs like Repeat Failure Rate and Time to Intervention.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-CAPA-001–085: the four issue-creation sources, triage and assignment, CAPA steps, five root cause methods, verification and reopening, SLA ladders, UX states, and edge cases. |
| [high-level-design.md](./high-level-design.md) | The Issue/CAPA Service boundary and component diagram, data ownership, events in/out, scale design (per-issue timer workflows, burst creation), multi-tenancy, failure modes, and technology choices. |
| [low-level-design.md](./low-level-design.md) | Postgres schema (`issues`, `capa_steps`, `root_cause_analyses`, `effectiveness_verifications`, `issue_events`), REST endpoints, sequence diagrams, SLA and repeat-failure pseudocode, caching, concurrency, and background jobs. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: component view, the golden temperature-exception trace, the SLA escalation ladder, the entity-relationship diagram, and the canonical issue state machine. |
