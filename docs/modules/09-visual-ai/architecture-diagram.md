# Visual AI — Architecture Diagrams

Focused Mermaid diagrams for the Visual AI module (EPIC-10). See `high-level-design.md` and
`low-level-design.md` for full narrative and schema detail.

## 1. Component / Container Diagram

```mermaid
flowchart TD
    subgraph CAPTURE["Frontline Capture"]
        MOB["Mobile App<br/>(camera-first evidence capture)"]
    end

    subgraph EVIDENCE["Evidence Module"]
        EVS["Evidence Service<br/>(S3-backed media, hashing, retention)"]
    end

    subgraph BACKBONE["Event Backbone (Kafka)"]
        TOPIC1["evidence.uploaded"]
        TOPIC2["visualai.inference.requested"]
        TOPIC3["visualai.result_ready /<br/>visualai.review_completed"]
    end

    subgraph VAI["Visual AI Service"]
        WORKER["Inference Queue Workers<br/>(autoscaled)"]
        CFGCACHE["UseCaseConfig Cache (Redis)"]
        RCB["Result Contract Builder"]
        ROUTER["Confidence Router"]
        RQ["Human Review Queue"]
        APPEAL["Appeals Workflow"]
        FEED["Feedback Capture / Export"]
        METRICS["Precision/Recall Rollup"]
        DB[("Postgres:<br/>use_case_configs, visual_ai_results,<br/>violations, review_decisions,<br/>model_versions")]
    end

    subgraph AIRUNTIME["AI Runtime Layer (Agentic Ops)"]
        GATEWAY["Model-Provider-Agnostic Gateway"]
        BACKEND1["Vision Model Backend A"]
        BACKEND2["Custom Classifier Backend B"]
        TRAINPIPE["Model Governance /<br/>Training Pipeline"]
        VIAGENT["Visual Inspection Agent<br/>(L1 Recommend)"]
    end

    subgraph CONSUMERS["Downstream Modules"]
        WORK["Work Orchestration"]
        ISSUES["Issues & CAPA"]
        ANALYTICS["Analytics & Control Tower"]
    end

    MOB -->|"capture photo/video"| EVS
    EVS -->|"evidence.uploaded"| TOPIC1
    TOPIC1 --> TOPIC2
    TOPIC2 --> WORKER
    WORKER <--> CFGCACHE
    WORKER --> RCB
    WORKER -->|"inference call"| GATEWAY
    GATEWAY --> BACKEND1
    GATEWAY --> BACKEND2
    RCB --> ROUTER
    ROUTER -->|"AUTO_PASSED/AUTO_FAILED"| DB
    ROUTER -->|"PENDING_REVIEW"| RQ
    RQ -->|"reviewer decision"| DB
    DB --> APPEAL
    APPEAL --> DB
    DB --> FEED
    FEED -->|"labeled dataset (per-tenant partitioned)"| TRAINPIPE
    DB --> METRICS
    VIAGENT -.->|"typed tool calls: get result, request re-inspection"| DB
    DB -->|"visualai.result_ready / review_completed"| TOPIC3
    TOPIC3 --> WORK
    TOPIC3 --> ISSUES
    TOPIC3 --> ANALYTICS
```

**Narration.** The Visual AI Service sits between the Evidence module (which owns the media) and
three consumer domains (Work Orchestration, Issues & CAPA, Analytics). Inference is deliberately
asynchronous and queue-mediated so that ingestion bursts never directly load the model backends,
and the model gateway is abstracted through the platform's shared AI Runtime layer so backends are
swappable per use case. The Visual Inspection Agent is a peer consumer of this module's data and
APIs, not a component inside it — it reasons about `VisualAiResult` history at its own configured
autonomy level and never bypasses the confidence router.

## 2. Sequence Diagram — Inference and Routing

```mermaid
sequenceDiagram
    participant EVS as Evidence Service
    participant Q as Inference Queue
    participant W as Inference Worker
    participant GW as Model Gateway
    participant DB as Postgres
    participant RQ as Review Queue

    EVS->>Q: evidence.uploaded (evidence_id, use_case)
    Q->>W: dispatch
    W->>GW: score(image, use_case_config)
    GW-->>W: compliance_score, confidence, violations[]
    W->>DB: persist VisualAiResult + Violation rows
    W->>W: apply confidence-threshold routing
    alt confidence high, no autofail-eligible violation
        W->>DB: status = AI_RESULT_AUTO_PASSED
    else confidence high, autofail-eligible violation, auto_fail_enabled
        W->>DB: status = AI_RESULT_AUTO_FAILED
    else otherwise
        W->>DB: status = AI_RESULT_PENDING_REVIEW
        W->>RQ: enqueue for human review
    end
    W->>EVS: (via event) visualai.result_ready
```

**Narration.** Routing always resolves to exactly one of three outcomes, and the low-confidence
path to `AI_RESULT_PENDING_REVIEW` is the safe default — it is reached whenever thresholds are not
clearly met in either direction, and always when `auto_fail_enabled` is false for the use case.
This is the mechanism behind the §24.1 acceptance test: a low-confidence result never auto-fails.

## 3. Sequence Diagram — Review and Feedback Loop

```mermaid
sequenceDiagram
    participant MGR as Store Manager
    participant API as Visual AI API
    participant DB as Postgres
    participant ISSUES as Issues & CAPA
    participant FEED as Feedback Export Job
    participant TRAIN as Model Governance (Agentic Ops)

    MGR->>API: review decision (ACCEPT / OVERRIDE / REJECT)
    API->>DB: insert ReviewDecision, update Violation(s), transition result to AI_RESULT_REVIEWED
    alt HIGH-severity violation confirmed
        API->>ISSUES: request Issue creation
        ISSUES-->>DB: Violation.issue_id set (cross-reference)
    end
    Note over FEED: nightly
    FEED->>DB: read new ReviewDecisions since last watermark
    FEED->>TRAIN: export labeled examples, partitioned per tenant (never cross-tenant)
```

**Narration.** Every reviewer decision does double duty: it corrects the compliance record shown
to the business (feeding Issues/CAPA when a HIGH violation is confirmed) and it becomes a labeled
training example for the model that produced the original result. The export is intentionally
batch and tenant-partitioned — no tenant's corrections ever influence another tenant's model unless
that tenant has explicitly opted into a shared program.

## 4. ER Diagram

```mermaid
erDiagram
    USE_CASE_CONFIGS ||--o{ VISUAL_AI_RESULTS : configures
    MODEL_VERSIONS ||--o{ USE_CASE_CONFIGS : "pinned by"
    MODEL_VERSIONS ||--o{ VISUAL_AI_RESULTS : "scored by"
    VISUAL_AI_RESULTS ||--o{ VIOLATIONS : contains
    VISUAL_AI_RESULTS ||--o{ REVIEW_DECISIONS : "reviewed via"

    USE_CASE_CONFIGS {
        uuid id PK
        uuid tenant_id
        text use_case
        integer config_version
        numeric auto_pass_confidence_threshold
        numeric auto_fail_confidence_threshold
        boolean auto_fail_enabled
    }
    MODEL_VERSIONS {
        uuid id PK
        text use_case
        text version_label
        text rollout_stage
        numeric production_precision
        numeric production_recall
    }
    VISUAL_AI_RESULTS {
        uuid id PK
        uuid tenant_id
        uuid evidence_id
        integer compliance_score
        numeric confidence
        text status
        bigint version
    }
    VIOLATIONS {
        uuid id PK
        uuid visual_ai_result_id FK
        text type
        text severity
        numeric confidence
        integer bounding_box
    }
    REVIEW_DECISIONS {
        uuid id PK
        uuid visual_ai_result_id FK
        text decision_type
        uuid reviewer_id
    }
```

**Narration.** `VisualAiResult` is the aggregate root of the module. `Violation` rows are
first-class children (not an embedded blob) so each can be independently reviewed, cross-referenced
to a CAPA `Issue`, and used as a discrete labeled training example. `ReviewDecision` is append-only
audit history against a result; a result can accumulate more than one (dual review, then an appeal
resolution) without losing the original.

## 5. State Diagram — VisualAiResult Lifecycle

```mermaid
stateDiagram-v2
    [*] --> AI_RESULT_PENDING: evidence.uploaded consumed, inference dispatched
    AI_RESULT_PENDING --> AI_RESULT_FAILED_PROCESSING: model backend timeout/outage,\nno usable config, or quality-gate rejection
    AI_RESULT_PENDING --> AI_RESULT_SCORED: model backend returns result contract
    AI_RESULT_FAILED_PROCESSING --> AI_RESULT_PENDING: retry (backend recovers /\nconfig configured / retaken photo)

    AI_RESULT_SCORED --> AI_RESULT_AUTO_PASSED: confidence >= auto_pass_threshold\nand no autofail-eligible violation
    AI_RESULT_SCORED --> AI_RESULT_AUTO_FAILED: confidence >= auto_fail_threshold\nand auto_fail_enabled\nand autofail-eligible violation present
    AI_RESULT_SCORED --> AI_RESULT_PENDING_REVIEW: thresholds not clearly met\n(safe default; also source-spec's\nAI_RESULT_UNREVIEWED alias)

    AI_RESULT_PENDING_REVIEW --> AI_RESULT_REVIEWED: single reviewer decision\n(ACCEPT/OVERRIDE/REJECT), dual review\nnot required or both reviewers agree
    AI_RESULT_PENDING_REVIEW --> AI_RESULT_NEEDS_ADJUDICATION: dual review required\nand reviewers disagree
    AI_RESULT_NEEDS_ADJUDICATION --> AI_RESULT_REVIEWED: higher-scope adjudicator resolves

    AI_RESULT_AUTO_PASSED --> AI_RESULT_APPEALED: appeal filed within window
    AI_RESULT_AUTO_FAILED --> AI_RESULT_APPEALED: appeal filed within window
    AI_RESULT_REVIEWED --> AI_RESULT_APPEALED: appeal filed within window

    AI_RESULT_APPEALED --> AI_RESULT_RESOLVED: resolver UPHOLDs or OVERTURNs

    AI_RESULT_AUTO_PASSED --> [*]: appeal window expires unused
    AI_RESULT_AUTO_FAILED --> [*]: appeal window expires unused
    AI_RESULT_REVIEWED --> [*]: appeal window expires unused
    AI_RESULT_RESOLVED --> [*]: terminal
```

**Narration.** The lifecycle has two safety properties by construction: (1) nothing reaches a
decided state (`AUTO_PASSED`/`AUTO_FAILED`) without first passing through `SCORED` and the
confidence router — there is no path that skips routing; and (2) every decided or reviewed state
remains appealable within its window, and an appeal always resolves to `RESOLVED`, never silently
disappearing. `AI_RESULT_UNREVIEWED` from the source-spec §12.2 contract is retained as the
external/legacy alias for whichever of `AUTO_PASSED`/`AUTO_FAILED`/`PENDING_REVIEW` a result is
before any human has touched it (see functional-spec.md §14 Resolved Decisions).
