# Module 09 — Visual AI

> This module turns photos that frontline workers already capture — a shelf, a floor, safety gear, signage — into structured, explainable compliance checks. AI scores each image, but only confident findings count without a human: everything uncertain goes to a manager review queue, and stores can appeal decisions. It is the platform's "policy before AI" principle applied to computer vision.

| | |
|---|---|
| **Module code** | `09-visual-ai` |
| **Owning service** | Visual AI Service (orchestration + result store; it hosts no models itself) |
| **Epics** | EPIC-10 (Visual AI) |

## In plain terms
- Configure what "good" looks like per use case: planogram/shelf compliance, promotional signage, brand execution, cleanliness, PPE/safety (protective gear), asset condition, or vehicle/site condition.
- Every matching photo gets a structured result: a 0–100 compliance score, an overall confidence, and a list of violations with type, severity, and where each one sits in the image (bounding boxes).
- Confidence decides the route: high-confidence results can auto-pass (and, once a tenant has earned it, auto-fail); anything uncertain goes to human review — low confidence never auto-fails.
- Managers review flagged images with the violation boxes drawn on them, then accept, override field-by-field, or reject the photo for a retake.
- Stores can appeal a decided result once within a set window; a higher-level manager upholds or overturns the original call.
- Every result records exactly which model version scored it; reviewer corrections feed precision/recall metrics and become labeled training data — kept strictly separate per tenant.

## Key concepts
| Concept | What it means |
|---|---|
| UseCaseConfig | The per-tenant, per-use-case setup: which model backend to call, confidence thresholds, violation taxonomy, and severity map; versioned, and disabled until an admin activates it. |
| VisualAiResult | The scored outcome for one photo — compliance score, confidence, status, violations — permanently pinned to the exact model and config versions that produced it. |
| Violation | One detected problem stored as its own record (type, severity, confidence, bounding box), so it can be reviewed, linked to a CAPA issue, and used as training data individually. |
| Confidence routing | The auto-pass / auto-fail / human-review thresholds; auto-fail is off by default and only allowed after precision ≥ 0.90 and recall ≥ 0.85 on HIGH-severity violations. |
| ReviewDecision | The human verdict (accept, override, reject — or appeal resolution); downstream modules see the reviewer's final call, while the original AI output is preserved for audit. |
| Appeal | A single, window-limited challenge (default 7 days) to a decided result, resolved by the location manager or the next org level up. |
| Model version traceability | Every result stores model backend/version and config version; new model versions roll out via shadow scoring on live traffic before cutover. |
| Labeled feedback export | Reviewer corrections collected as training examples for the model-training pipeline, partitioned so one tenant's data never trains another tenant's model. |

## How it connects to the rest
- Triggered by **Evidence (05)**: it consumes the `evidence.uploaded` event and reads images via signed URLs; it never owns the media itself.
- Feeds **Issues & CAPA (06)**: a confirmed HIGH-severity violation on a failed result can automatically open an Issue, and an overturned appeal flags that issue for review.
- Works with **Work Orchestration (02)**: form fields declare which visual AI use case applies, results can gate a work item's `VERIFIED` transition, and review SLAs reuse its escalation engine.
- Depends on **Agentic Ops (10)** for the model-provider gateway, model governance, and the Visual Inspection Agent (which consumes results as a typed tool); feeds **Analytics (11)** compliance trends and the AI Override Rate KPI.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-VAI-001–076: the result contract field by field, per-use-case configuration, confidence routing, review and appeals workflows, model traceability, precision/recall and the retraining loop, UX states, and edge cases. |
| [high-level-design.md](./high-level-design.md) | The Visual AI Service as an orchestration and result store (no model hosting), component diagram, data ownership, queue-based burst handling, per-tenant training-data isolation, fail-open failure modes, and technology choices. |
| [low-level-design.md](./low-level-design.md) | Postgres schema (`use_case_configs`, `visual_ai_results`, `violations`, `review_decisions`, `model_versions`), REST endpoints, sequence diagrams (inference routing, review-to-feedback-to-issue creation, shadow model rollout), routing and precision/recall pseudocode, caching, concurrency, jobs. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: component view from capture to feedback, the inference-and-routing sequence, the review/feedback loop sequence, the entity-relationship diagram, and the VisualAiResult lifecycle state machine. |
