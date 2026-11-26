# Employee Engagement & Voice of Frontline — Architecture Diagrams

Module: `18-employee-engagement`

## 1. Component Diagram

```mermaid
flowchart TD
    subgraph Clients
        Mobile[Mobile App]
        Web[Web Admin]
    end

    BFF[API Gateway / BFF]

    subgraph EngagementService [Engagement Service]
        SurveyAPI[Survey Campaign API]
        ResponseAPI[Response Collection API]
        Anon[[Anonymization Boundary]]
        RecogAPI[Recognition API]
        AggEngine[Aggregation Engine]
        ScoreEngine[Score Rollup Engine]
    end

    subgraph Data [engagement schema]
        Surveys[(pulse_surveys /\nsurvey_questions)]
        Responses[(survey_responses)]
        Dedupe[(respondent_dedupe_tokens)]
        Aggregates[(survey_response_aggregates)]
        Recog[(recognition_events)]
        Scores[(engagement_scores)]
    end

    Comms07[Communications & Knowledge\naudience resolution + delivery]
    WorkOrch02[Work Orchestration\nSurvey WorkItem type]
    Gam22[Gamification\npoints ledger]
    Analytics11[Analytics & Control Tower]

    Mobile --> BFF --> SurveyAPI & ResponseAPI & RecogAPI
    Web --> BFF

    SurveyAPI --> Comms07
    SurveyAPI --> WorkOrch02
    SurveyAPI --> Surveys

    ResponseAPI --> Anon --> Responses
    Anon --> Dedupe

    RecogAPI --> Recog
    RecogAPI -.event.-> Gam22

    AggEngine --> Responses
    AggEngine --> Aggregates
    ScoreEngine --> Aggregates
    ScoreEngine --> Recog
    ScoreEngine --> Scores

    Web -- threshold-gated reads --> AggEngine
    Web -- threshold-gated reads --> ScoreEngine

    SurveyAPI -.events.-> Analytics11
    ScoreEngine -.events.-> Analytics11
```

**Narration**: The Engagement Service has three write paths (survey design, response collection,
recognition) and two read paths (aggregation, score rollup), and every read path is structurally
separated from raw `survey_responses` — only the Aggregation Engine's background worker ever queries
that table, and even it writes only threshold-annotated results forward. Anonymity is preserved
end-to-end here because the component that *knows* who submitted (the `Anon` boundary, transiently,
during dedupe-token derivation) is a different component from every component that *reports*
results, and no data path connects the two: the boundary throws away the `user_id` before the
response row is ever persisted, and it holds the only key capable of reversing dedupe tokens back to
users, which no other component in this diagram has access to.

## 2. Sequence Diagram — Survey Publish to Response Collection

```mermaid
sequenceDiagram
    participant Admin as Ops Admin
    participant Eng as Engagement Service
    participant Comms as Communications & Knowledge
    participant WI as Work Orchestration
    participant Mobile as Frontline Associate

    Admin->>Eng: Publish survey campaign (anonymity_mode=ANONYMOUS)
    Eng->>Comms: resolve_audience(rules)
    Comms-->>Eng: recipient user_ids
    Eng->>WI: create Survey WorkItem per recipient
    Eng->>Comms: deliver "survey available" notification
    Comms-->>Mobile: push notification
    Mobile->>Eng: Submit response
    Note over Eng: user_id consumed only to derive a\none-way dedupe token, then discarded —\nnever written alongside the response
    Eng-->>Mobile: Confirmation — "submitted anonymously"
```

**Narration**: The audience-resolution and delivery hops are entirely delegated to Communications &
Knowledge — this module never learns how to fan out a notification or evaluate an audience rule; it
only asks Communications for a recipient list and hands it a notification request. Anonymity is
preserved at the last hop: when the associate submits, their identity is used exactly once, inside
the service, to compute a dedupe token, and is never stored next to the answers themselves.

## 3. Sequence Diagram — Anonymity-Safe Aggregation Read

```mermaid
sequenceDiagram
    participant Mgr as Manager / HR / Admin (any scope breadth)
    participant Eng as Engagement Service
    participant RBAC as Authorization Service
    participant Agg as survey_response_aggregates

    Mgr->>Eng: GET results for group_key = "Store #12"
    Eng->>RBAC: is Mgr authorized for this node? (scope gate)
    RBAC-->>Eng: yes (RBAC only answers "may ask," not "may see stats")
    Eng->>Agg: fetch precomputed respondent_count + stats
    alt respondent_count < aggregation_threshold
        Agg-->>Eng: count=3, stats present
        Eng-->>Mgr: suppressed=true (raw count withheld)
    else respondent_count >= aggregation_threshold
        Agg-->>Eng: count=41, stats present
        Eng-->>Mgr: suppressed=false, full stats
    end
```

**Narration**: This is the diagram that makes the anonymity guarantee concrete: RBAC and the
aggregation threshold are two independent, sequential gates, and **broad scope never substitutes for
group size** — the same requester gets full data for a large store and a suppressed response for a
small one, in the same session, because the threshold check runs after authorization and depends
only on `respondent_count`, a value RBAC has no opinion on. The raw count itself is withheld even in
the suppressed branch, closing the "3 of 5" inference gap.

## 4. ER Diagram — Owned Entities

```mermaid
erDiagram
    PULSE_SURVEYS ||--o{ SURVEY_QUESTIONS : has
    PULSE_SURVEYS ||--o{ PULSE_SURVEY_RUNS : "scheduled as"
    PULSE_SURVEY_RUNS ||--o{ SURVEY_RESPONSES : collects
    PULSE_SURVEY_RUNS ||--o{ RESPONDENT_DEDUPE_TOKENS : dedupes
    PULSE_SURVEY_RUNS ||--o{ SURVEY_RESPONSE_AGGREGATES : "aggregates into"
    RECOGNITION_EVENTS }o--|| PULSE_SURVEYS : "independent (not FK-linked)"

    PULSE_SURVEYS {
        uuid id PK
        string anonymity_mode
        int aggregation_threshold
        string status
    }
    SURVEY_RESPONSES {
        uuid id PK
        uuid user_id "NULL if anonymous"
        jsonb response_value
    }
    RESPONDENT_DEDUPE_TOKENS {
        uuid id PK
        string respondent_token "one-way HMAC, no user_id"
    }
    SURVEY_RESPONSE_AGGREGATES {
        uuid id PK
        int respondent_count
        numeric mean_value
    }
    RECOGNITION_EVENTS {
        uuid id PK
        uuid from_user_id
        uuid to_user_id
    }
    ENGAGEMENT_SCORES {
        uuid id PK
        uuid node_id
        numeric composite_score
        bool is_suppressed
    }
```

**Narration**: `SURVEY_RESPONSES` deliberately has no relationship drawn to any identity table in
this diagram for anonymous runs (`user_id` is nullable and null in that mode); `RESPONDENT_DEDUPE_
TOKENS` has no foreign key to `SURVEY_RESPONSES` at all — the two tables are joined by neither a key
nor application logic, which is what makes "duplicate prevention without identity storage"
structurally true rather than merely a policy promise. `RECOGNITION_EVENTS` is shown unlinked to the
survey subsystem because recognition is intentionally attributed (never anonymous) and must not share
a data model with the anonymity-protected tables.

## 5. State Diagram — PulseSurvey Campaign Lifecycle

```mermaid
stateDiagram-v2
    [*] --> DRAFT
    DRAFT --> SCHEDULED : publish (publish_at in future)
    DRAFT --> ACTIVE : publish (publish_at = now)
    SCHEDULED --> ACTIVE : scheduler fires at publish_at
    ACTIVE --> CLOSED : close_at reached, or force-close by admin
    CLOSED --> RESULTS_PUBLISHED : final aggregation completes

    DRAFT --> DRAFT : edit questions / anonymity_mode (mutable pre-schedule)
    note right of DRAFT
        anonymity_mode is the last field
        that can change here — it locks
        the moment status leaves DRAFT
    end note

    note right of ACTIVE
        Responses accepted; each response
        passes through the Anonymization
        Boundary before persistence
    end note

    note right of CLOSED
        No further responses accepted;
        final full-recompute aggregation
        runs against every group_key
        observed in this run
    end note

    note right of RESULTS_PUBLISHED
        Dashboards serve only from
        survey_response_aggregates from
        this point on — raw responses
        are never queried by any reporting path
    end note

    RESULTS_PUBLISHED --> [*]
```

**Narration**: Anonymity is preserved across every transition, not just at submission time. The
`anonymity_mode` decision is frozen the instant a survey leaves `DRAFT` (FR-ENG-005), so respondents
who see "this survey is anonymous" during `ACTIVE` can trust that promise was fixed before delivery,
not adjustable after responses start arriving. At `CLOSED → RESULTS_PUBLISHED`, the system performs
its last full aggregate computation and from then on every downstream consumer — manager dashboards,
Analytics, HR reporting — reads exclusively from the threshold-annotated aggregate table, never from
`survey_responses`, which is the structural guarantee that outlives the campaign itself.
