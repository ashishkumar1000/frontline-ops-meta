# Food Safety & HACCP — Architecture Diagrams

Focused Mermaid diagrams for this module only. See `high-level-design.md` and
`low-level-design.md` for full narrative and schema detail.

## 1. Component / Container Diagram

```mermaid
flowchart TD
    subgraph HACCP [Food Safety and HACCP - configuration/orchestration layer]
        PLANBUILDER[HACCP Plan Builder]
        BINDER[CCP Binding Translator]
        PROJECTOR[Compliance Read-Model Projector]
        BREACHFRONT[Food-Safety Corrective Action Front End]
        FALLBACK[Sensor-Offline Fallback Orchestrator]
    end

    QA[QA / Food Safety Admin] --> PLANBUILDER
    PLANBUILDER --> BINDER
    BINDER -- "author TemplateVersion" --> FORMS[Forms and Workflow Builder]
    BINDER -- "author SensorAlertRule" --> IOT[IoT Sensor Platform]

    WORK[Work Orchestration]
    ASSOC[Frontline Associate] --> WORK --> FORMS
    SENSOR[Sensor Gateway] --> IOT

    FORMS -- "Submission, ccp_id tagged" --> CAPA[Issues and CAPA]
    IOT -- "breach: Issue, ccp_id tagged" --> CAPA
    IOT -- "sensor.offline" --> FALLBACK
    FALLBACK -- "ad-hoc manual check" --> WORK

    CAPA --> BREACHFRONT
    BREACHFRONT -- "FoodSafetyCorrectiveAction" --> CAPA
    BREACHFRONT --> EVID[Evidence]

    FORMS -- "submission.created" --> PROJECTOR
    IOT -- "sensor.alert.*, sensor.health.*" --> PROJECTOR
    CAPA -- "issue.resolved" --> PROJECTOR
    PROJECTOR --> READMODEL[(ccp_monitoring_records)]
    READMODEL --> ANALYTICS[Analytics and Control Tower]
    READMODEL --> EXPORT[Audit Export]
    MGR[Store/Site Manager] --> READMODEL
    AGENT[Compliance Monitor Agent] --> CAPA
```

This module owns five small, purely orchestration/configuration components — no ingestion, no
form-rendering, no exception-lifecycle logic. The Binding Translator is the fork point (manual vs.
automated); everything after a breach is detected flows through the same Issues & CAPA surface
regardless of which path produced it.

## 2. Sequence Diagram — Golden Path: Automated CCP Breach to Corrective Action

```mermaid
sequenceDiagram
    participant Sensor as Sensor Gateway
    participant IoT as IoT Sensor Platform
    participant Capa as Issues and CAPA
    participant HACCP as Food Safety and HACCP
    participant Mgr as Store Manager

    Sensor->>IoT: reading (45.2F, walk-in cooler)
    IoT->>IoT: Sustained-duration rule breach (>41F for 15 min)
    IoT->>Capa: POST /internal/issues (ccp_id-tagged)
    Capa-->>IoT: 201 { issue_id }
    IoT->>HACCP: sensor.alert.triggered (projected into ccp_monitoring_records)
    Capa->>Mgr: notify: corrective action required
    Mgr->>HACCP: record corrective action (DISCARD_PRODUCT + photo evidence)
    HACCP->>Capa: complete CORRECTIVE_ACTION CAPA step
    Capa->>Capa: CORRECTIVE_ACTION -> VERIFICATION -> RESOLVED
```

This is the platform's Appendix A golden scenario, formalized under a named CCP: the breach and
Issue mechanics are entirely IoT Sensor Platform's and Issues & CAPA's existing machinery; this
module supplies the CCP tag and the food-safety-worded corrective-action capture that satisfies the
same CAPA gate a generic Issue would require anyway.

## 3. Sequence Diagram — Manual CCP Check with Blocking Breach Sub-Flow

```mermaid
sequenceDiagram
    participant Assoc as Frontline Associate
    participant Forms as Forms and Workflow Builder
    participant Capa as Issues and CAPA
    participant HACCP as Food Safety and HACCP

    Assoc->>Forms: Enter temperature reading (out of range)
    Forms->>Forms: Calculated field flags out-of-range
    Forms->>Forms: corrective_action becomes mandatory (submit blocked)
    Assoc->>Forms: Complete corrective action + evidence
    Assoc->>Forms: Submit (now allowed)
    Forms->>Capa: POST /internal/issues (ccp_id-tagged, FORM_TRIGGERED)
    Forms->>HACCP: submission.created (projected)
```

The blocking gate is Forms' own conditional-mandatory validation (Module 03) — this module never
implements a second, competing validation layer; it only configures the field logic at
CCP-authoring time and reads the resulting submission afterward.

## 4. ER Diagram — Owned Entities

```mermaid
erDiagram
    HACCP_PLANS ||--o{ HACCP_PLAN_VERSIONS : "has versions"
    HACCP_PLAN_VERSIONS ||--o{ CRITICAL_CONTROL_POINTS : defines
    CRITICAL_CONTROL_POINTS ||--o{ CCP_MONITORING_RECORDS : "monitored by"
    CRITICAL_CONTROL_POINTS ||--o{ FOOD_SAFETY_CORRECTIVE_ACTIONS : "breach actions"
    CCP_MONITORING_RECORDS ||--o| FOOD_SAFETY_CORRECTIVE_ACTIONS : "resolved by"

    HACCP_PLANS {
        uuid id PK
        text name
        uuid current_published_version_id
    }
    HACCP_PLAN_VERSIONS {
        uuid id PK
        uuid plan_id FK
        int version
        enum status
        jsonb process_flow
        jsonb hazard_analysis
    }
    CRITICAL_CONTROL_POINTS {
        uuid id PK
        uuid plan_version_id FK
        enum monitoring_method
        numeric critical_limit_value
        uuid bound_template_version_id
        uuid bound_sensor_alert_rule_id
    }
    CCP_MONITORING_RECORDS {
        uuid id PK
        uuid ccp_id FK
        enum source_type
        enum compliance_status
        uuid issue_id
    }
    FOOD_SAFETY_CORRECTIVE_ACTIONS {
        uuid id PK
        uuid ccp_id FK
        uuid issue_id
        enum action_type
        uuid evidence_id
    }
```

`CCP_MONITORING_RECORDS` is a read-model table, not a system of record — it is populated entirely
from events produced by Forms and IoT Sensor Platform (LLD §1.3), which is why this diagram shows
no other module's tables even though the data ultimately originates elsewhere.

## 5. State Diagram — CCP Monitoring Record Lifecycle

```mermaid
stateDiagram-v2
    [*] --> DUE : WorkItem scheduled (manual) or continuous window opens (automated)
    DUE --> IN_COMPLIANCE : reading/submission within critical limit
    DUE --> BREACHED : reading/submission outside critical limit
    IN_COMPLIANCE --> [*] : record closed, no further action needed

    BREACHED --> CORRECTIVE_ACTION_RECORDED : FoodSafetyCorrectiveAction recorded\n(mandatory - cannot skip, FR-HACCP-044)
    BREACHED --> INCONCLUSIVE : sensor offline mid-window (automated only)\nEdge Case S5.1
    INCONCLUSIVE --> CORRECTIVE_ACTION_RECORDED : manager acknowledges + fallback manual\ncheck confirms status
    INCONCLUSIVE --> [*] : explicit manager acknowledgement without breach confirmed

    CORRECTIVE_ACTION_RECORDED --> VERIFIED : Issues and CAPA verification step passes\n(re-check or manager confirmation)
    CORRECTIVE_ACTION_RECORDED --> BREACHED : verification fails - Issue REOPENED,\nnew corrective action required
    VERIFIED --> [*] : record closed, included in shift review and audit export
```

`BREACHED` can never transition directly to `[*]` or to `VERIFIED` — every path out of a breach
passes through `CORRECTIVE_ACTION_RECORDED` first, which is the state-machine expression of
FR-HACCP-044 ("cannot skip past a failed CCP"). `VERIFIED` is reached only via Issues & CAPA's own
verification step (Module 06), reflecting that this module never grants itself authority to close
out a breach on its own — verification remains that module's gate, reused unchanged.
