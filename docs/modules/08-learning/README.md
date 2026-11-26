# Module 08 — Learning

> This module connects training to actual operational results. When the same check keeps failing, it can automatically assign targeted microlearning to the person involved, require them to pass a knowledge check, then re-inspect the original check to measure whether the failure actually stopped. Learners see assignments on their Today screen (including fully offline); managers see completion, quiz failures, and certification lapses for their team.

| | |
|---|---|
| **Module code** | `08-learning` |
| **Owning service** | Learning Service |
| **Epics** | EPIC-09 (Learning) |

## In plain terms
- Assign training three ways: by hand, by role (so new hires in a role automatically get it), or automatically when a failure pattern repeats — the failure-triggered remediation path.
- Complete courses on mobile — documents and videos work fully offline — and finish by passing a knowledge-check quiz (default 80% to pass, up to 3 attempts).
- Earn certifications that expire: renewal reminders fire before expiry, and — if the tenant sets it — new safety-critical work is blocked when a required certification has lapsed.
- After a failure-triggered course, the system schedules a re-inspection of the exact check that kept failing and records whether it now passes (measured improvement).
- If the re-inspection still fails, the case escalates back to Issues & CAPA — the gap is probably process or equipment, not knowledge.
- The Training Agent (AI) proposes remediation assignments, but at the default autonomy level a manager approves each one before it is assigned.

## Key concepts
| Concept | What it means |
|---|---|
| Course / CourseVersion | A logical course and its immutable published versions (the platform's Template/TemplateVersion pattern); an in-progress assignment always completes against the version it was assigned. |
| LearningAssignment | The central record — `ASSIGNED → IN_PROGRESS → COMPLETED → VERIFIED/FAILED` — with an origin of `MANUAL`, `ROLE_STANDING`, or `FAILURE_TRIGGERED`. |
| Knowledge check (QuizAttempt) | The quiz attached to a course version; passing it is required to complete, and exhausting attempts fails the assignment and surfaces it to the manager as an exception. |
| Certification / CertificationRecord | A renewable credential with an expiry date and an append-only history per user; each certification sets an expiry policy of `REPORT_ONLY`, `WARN`, or `BLOCK_NEW_WORK`. |
| FailureToCourseMapping | An admin-maintained rule table mapping failure categories to courses; the Training Agent reads it to pick the right remediation. |
| Repeated-failure loop | The module's core loop: repeated failure → microlearning → knowledge check → re-inspection → measured improvement (default triggers: 3 failures in 30 days at one location, or 2 by the same user). |
| OutcomeMeasurement | The before/after comparison that proves (or disproves) training fixed the failing check — did the re-inspection pass where the original occurrence failed. |
| Training Agent | The AI agent that proposes failure-triggered assignments; default autonomy is L1 Recommend, meaning a manager confirms before anything is assigned. |

## How it connects to the rest
- Consumes **Issues & CAPA (06)** signals (`issue.resolved`, repeat-failure patterns) as assignment triggers, and escalates back there when a re-inspection shows training did not help.
- Projects every assignment into **Work Orchestration (02)** as a `TRAINING_ASSIGNMENT` WorkItem for Today-screen visibility and due dates; Work Orchestration in turn calls Learning to block work claims when a certification has lapsed.
- Produces outcome facts for **Analytics (11)** (the Training-to-outcome-improvement KPI) and sends reminders through **Communications / Notification Hub (07)**.
- Runs its Training Agent on **Agentic Ops / Agent Runtime (10)**, may link courses to Knowledge documents (07), and rides on **Mobile Offline (04)** for offline content caching and idempotent completion sync.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | MVP/Phase-2-tagged requirements FR-LEARN-001–066: course authoring, quiz mechanics, certification, the three assignment models, the repeated-failure loop, offline access, plus NFRs and firm resolved decisions. |
| [high-level-design.md](./high-level-design.md) | The Learning Service component diagram, data ownership (links to issues/work items rather than copying them), events in/out, 100k-user assignment fan-out, CDN video delivery, failure modes, and technology choices. |
| [low-level-design.md](./low-level-design.md) | Postgres schema (courses, assignments, quiz attempts, certifications, outcome measurements, failure-to-course mappings), REST endpoints, sequence diagrams (failure-triggered assignment, offline sync, expiry reminders), trigger and improvement-measurement pseudocode, caching, jobs, errors. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: component view, failure-triggered assignment sequence, offline completion sync sequence, entity-relationship diagram, and the LearningAssignment lifecycle state machine. |
