# Module 03 — No-Code Forms & Workflow Builder

> Lets an Operations Admin turn an operating standard — an opening checklist, a temperature audit, an inspection, a survey — into a digital form with logic ("if the freezer reads above 8°C, a corrective action becomes required") without writing code, then version and publish it safely. Frontline associates and auditors fill the published forms on mobile, online or offline. Every completed capture becomes a Submission this module owns.

| | |
|---|---|
| **Module code** | FORM |
| **Owning service** | Form Service |
| **Epics** | EPIC-04 |

## In plain terms
- Build forms from a rich field library — text, numbers, yes/no, matrix grids, barcode/QR scans, GPS, photos/video/audio, signatures, document uploads, lookups, repeating groups, and auto-calculated fields.
- Add logic without code: fields appear or become mandatory based on other answers, formulas compute totals, risk scores weight answers ("cold_chain_risk = HIGH → require corrective action, create an issue, notify the store manager"), and answers branch the flow.
- Preview the blast radius before publishing: Impact Preview shows how many locations, work items, and users a change affects, and can simulate the new rules against recent real submissions.
- Publish immutable, hash-stamped versions — a form filled offline always completes against the exact version it started with, and a newer publish never rewrites history.
- Pilot a new version to a few locations first, then roll out; capture submissions idempotently so a retried sync never creates duplicates.
- Go beyond plain fields with composable cards, buttons that raise dialogs, and forwarding that hands off to a different template entirely.

## Key concepts
| Concept | What it means |
|---|---|
| Template | The stable identity of a form (e.g. "Daily Deli Checklist") that persists across versions. |
| TemplateVersion | One immutable-once-published snapshot of the form — schema, logic rules, and scoring — cryptographically pinned by a `schema_hash`. |
| Submission | One filling of one exact TemplateVersion; accepted exactly once thanks to a client-generated `client_uuid` idempotency key. |
| Risk score | A weighted, thresholded score computed from answers (bands like LOW/MEDIUM/HIGH) that can trigger mandatory fields, issue creation, or notifications. |
| Impact Preview | A pre-publish report: affected locations/work items/users, a structural diff against the prior version, and a simulation of the new rules. |
| Conditional logic | Rules (`rules_json`) for visibility, mandatory conditions, cross-field validation, and branching — the same engine runs on-device offline and server-side at ingestion. |
| Tier-1 / Tier-2 validation | Immediate on-device checks (Tier-1) plus authoritative server-side re-validation, including cross-record rules that compare against recent submissions. |
| Repeating group | A fillable sub-form the user can repeat at runtime (e.g. one block per pallet), with stable instance IDs so calculations stay deterministic. |

## How it connects to the rest
- Is scheduled-and-assigned by Work Orchestration (02): the Work Service decides when and to whom a form-bearing WorkItem goes; this module renders and validates the form itself.
- Is rendered on-device by the Mobile & Offline module (04), which caches published schemas (delivered via CDN) and runs the same rule engine locally.
- Marks fields as evidence-bearing and hands media off to Evidence Management (05) — it stores only evidence references and hashes, never the bytes.
- Publishes risk scores that can trigger Issues & CAPA (06) and photo fields consumed by Visual AI (09); the Agent Runtime's Workflow Builder Agent (10) drafts new TemplateVersions through the same gated publish path as humans.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-FORM-001–087: every field type, the full logic rule set, the Builder UX, the TemplateVersion lifecycle and immutability, Impact Preview, pilot rollout, and the card/dialog/cross-template-forwarding extension. |
| [high-level-design.md](./high-level-design.md) | The Form Service's compile → validate → hash → version pipeline, CDN-backed schema distribution for offline devices, bursty-submission ingestion design, failure modes, and the embedded-JSONB schema decision. |
| [low-level-design.md](./low-level-design.md) | Full Postgres schema (`templates`, `template_versions`, `submissions`, `submission_answers`, impact previews, submission chains), REST endpoints, the rule-evaluation engine, hashing, and caching strategy. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: component diagram, publish-with-Impact-Preview sequence, idempotent submission-ingestion sequence, cross-template forwarding flow, and the schema-distribution flow from Versioning Store to CDN to device. |
