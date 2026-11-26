# Module 20 — EHS — Safety Incident & Risk Register

> This module handles formal workplace safety: injuries, illnesses, near-misses, property damage,
> plus the risk assessments and hazard registers that prevent them. Its reason for existing is
> that a safety incident carries legal obligations a generic operational issue does not — it may
> have to be logged with a regulator within a fixed window (for example, OSHA-style
> "record within 7 days" rules in the US) and kept for years after the fix is done. Frontline
> associates report incidents, and EHS (Environment, Health & Safety) officers own the
> classification, deadlines, and records auditors will ask about later.

| | |
|---|---|
| **Module code** | EHS (`FR-EHS-###`) |
| **Owning service** | EHS Service |
| **Tier** | 2 |

## In plain terms
- Report an injury, illness, near-miss, or property-damage event from the mobile app in minutes —
  including anonymously for near-misses — with witness statements attached.
- Classify each incident against the tenant's jurisdiction's rules (e.g. OSHA-style
  `RECORDABLE` vs. `FIRST_AID_ONLY`), and reclassify later if the injury turns out to be more
  serious than first thought, with the full classification history preserved.
- Never miss a filing deadline: the system computes each applicable regulatory reporting deadline,
  counts down with escalating alerts, and flags missed deadlines as a fail-loud compliance event.
- Run a proactive safety program: score risks on a likelihood-by-severity matrix, keep a register
  of hazards with controls and review dates, and log positive/negative safety observations.
- Keep records as long as the law requires: safety incidents get a regulatory retention period
  (default recommendation 5 years) that overrides the platform's normal retention, and cannot be
  deleted by anyone — including tenant admins — while it runs.

## Key concepts
| Concept | What it means |
|---|---|
| `SafetyIncident` | One reportable safety event, with its own lifecycle (`REPORTED → CLASSIFIED → INVESTIGATING → CORRECTIVE_ACTION_LINKED → CLOSED`, plus a `REOPENED` branch on audit findings). |
| `IncidentWitnessStatement` | An append-only statement from someone who saw the incident (text, audio, video, or signed document). |
| `RiskAssessment` | A likelihood × severity score for one risk, plus residual risk after controls (the standard hierarchy-of-controls: elimination, substitution, engineering, administrative, PPE). |
| `HazardRegisterEntry` | A known hazard at a location or asset, its controls, its owner, and when it must next be reviewed. |
| `SafetyObservation` | A positive or negative at-risk behavior/condition observation, submitted via the `SAFETY_OBSERVATION` WorkItem type. |
| `RegulatoryTaxonomy` / `RegulatoryReportingRule` | Per-jurisdiction configuration that decides how incidents are classified and which filing deadlines apply — not hardcoded to one country. |
| `RegulatoryReportDeadline` | The tracked clock for one incident's required filing, with escalating alerts and a permanent `late` flag if missed. |

## How it connects to the rest
- **Issues & CAPA (module 06 / EPIC-07)** — the module is explicitly a thin regulatory layer *over*
  Issues & CAPA, not a parallel corrective-action engine: an incident spawns an `Issue` with
  `source = SAFETY_INCIDENT` for root-cause analysis and corrective actions, and listens for
  `issue.resolved` to close its own loop. It never re-implements CAPA steps.
- **Work Orchestration (module 02 / EPIC-03)** — this module is the functional owner of what
  happens when `Incident`/`SAFETY_OBSERVATION` WorkItems are submitted; scheduling and state
  machine stay with Work Orchestration.
- **Evidence (module 05 / EPIC-06)** — incident photos, witness recordings, and filed report
  documents are Evidence records by ID; this module stamps a retention override on those links.
- **Learning (module 08 / EPIC-09)** — hazard controls and corrective actions can carry training
  assignments for affected staff.
- **Visual AI (module 10 / EPIC-10)** and **Analytics (module 11 / EPIC-12/15)** — a PPE-violation
  detection can suggest (never auto-create) an observation or incident; TRIR/LTIR-style safety
  KPIs are computed in Analytics from this module's events.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-EHS-001–085 covering incident reporting, Issue linkage, risk assessments, hazard register, observations, regulatory deadlines, and retention. |
| [high-level-design.md](./high-level-design.md) | The "thin regulatory layer over Issues & CAPA" stance, component diagram, data ownership, and the polling-sweep choice for deadline tracking. |
| [low-level-design.md](./low-level-design.md) | Full schema (11 tables including append-only history tables), REST endpoints, sequence diagrams, and pseudocode for deadline calculation and risk scoring. |
| [architecture-diagram.md](./architecture-diagram.md) | Component diagram, incident-to-CAPA feedback-loop and deadline-escalation sequence diagrams, and the incident lifecycle state machine. |
