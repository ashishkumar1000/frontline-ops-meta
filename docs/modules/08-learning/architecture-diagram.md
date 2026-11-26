# Learning — Architecture Diagrams

Module corresponds to **EPIC-09 (Learning)**. Focused diagram set only; narrative detail lives in
functional-spec.md, high-level-design.md and low-level-design.md.

## 1. Component / Container Diagram

```mermaid
flowchart TD
    subgraph Clients
        MOB[Mobile App]
        WEB[Web Admin Console]
    end

    GW[API Gateway / BFF]

    subgraph LS[Learning Service]
        API[Learning API Layer]
        CCS[Course/Content Store]
        AE[Assignment Engine]
        QE[Knowledge-Check / Quiz Engine]
        CE[Certification Engine]
        OLA[Outcome-Linkage Analytics Feed]
        FCM[FailureToCourseMapping]
    end

    PG[(PostgreSQL)]
    OBJ[(Object Storage + CDN)]
    KAFKA[[Event Backbone]]
    TEMPORAL[Temporal]
    REDIS[(Redis)]

    WORK[Work Orchestration]
    ISSUES[Issues & CAPA]
    ANALYTICS[Analytics]
    AGENT[Agent Runtime<br/>Training Agent]
    NOTIF[Notification Hub]

    MOB --> GW --> API
    WEB --> GW
    API --> CCS & AE & QE & CE & REDIS
    CCS --> OBJ & PG
    AE --> PG & TEMPORAL
    QE --> PG
    CE --> PG & TEMPORAL
    OLA --> PG & KAFKA
    AE --> FCM
    AGENT -. schedule_training .-> AE
    AE -- training.assigned --> KAFKA
    QE -- training.completed --> KAFKA
    KAFKA -- issue.resolved --> AE
    AE --> WORK
    WORK -- work.verified --> OLA
    CE -- reminders --> NOTIF
    OLA --> ANALYTICS
```
The Learning Service is a single deployable owning five internal components: the course/content
store (authoring + versioning), the assignment engine (all three assignment origins), the
knowledge-check engine, the certification engine, and the outcome-linkage feed. It depends on
Postgres for transactional state, object storage/CDN for content, Temporal for expiry/reminder
timers, Kafka for domain events, and Redis for hot-path caching. It integrates with Work
Orchestration (WorkItem projection), Issues/CAPA and Analytics (failure signals in, outcome facts
out), the Agent Runtime (Training Agent's `schedule_training` tool target), and Notification Hub
(reminder delivery).

## 2. Sequence Diagram — Failure-Triggered Assignment

```mermaid
sequenceDiagram
    participant Issues as Issues & CAPA
    participant Kafka as Event Backbone
    participant Agent as Training Agent
    participant Learn as Learning Service
    participant Mgr as Manager (L1 approval)
    participant Work as Work Orchestration

    Issues->>Kafka: issue.resolved
    Kafka->>Agent: deliver event
    Agent->>Learn: get_recent_failures / lookup FailureToCourseMapping
    Learn-->>Agent: course + autonomy_level
    Agent->>Mgr: recommendation (L1 Recommend)
    Mgr->>Agent: approve
    Agent->>Learn: schedule_training(...)
    Learn->>Learn: dedupe check + create LearningAssignment
    Learn->>Kafka: training.assigned
    Kafka->>Work: create TRAINING_ASSIGNMENT WorkItem
```
A repeated failure (detected by Issues/CAPA or Analytics) reaches the Training Agent via the event
backbone. The agent maps the failure category to a course through Learning's
`FailureToCourseMapping` table, and — at the default L1 Recommend autonomy — surfaces the proposal
for manager approval before Learning creates the `LearningAssignment` and Work Orchestration
projects it as a `TRAINING_ASSIGNMENT` WorkItem.

## 3. Sequence Diagram — Offline Completion Sync

```mermaid
sequenceDiagram
    participant Device as Mobile Device
    participant API as Learning API
    participant DB as Postgres

    Note over Device: Offline: view content, attempt quiz locally
    Device->>Device: store content_viewed + quiz_attempt with client_uuid
    Note over Device: Reconnect
    Device->>API: POST /content-viewed {client_uuid}
    API->>DB: idempotent upsert
    API-->>Device: 200 OK, status=IN_PROGRESS
    Device->>API: POST /quiz-attempts {client_uuid}
    API->>DB: idempotent insert, server recomputes authoritative score
    API-->>Device: 201 Created, status=COMPLETED
    API->>API: emit training.completed
```
Content viewing and quiz attempts are recorded locally while offline, each tagged with a
`client_uuid`. On reconnect, both sync calls are idempotent — a duplicate replay returns the
original result — and the server always recomputes the authoritative quiz score rather than
trusting the client's locally computed value.

## 4. ER Diagram

```mermaid
erDiagram
    COURSES ||--o{ COURSE_VERSIONS : has
    COURSE_VERSIONS ||--o{ QUIZ_QUESTIONS : contains
    COURSE_VERSIONS ||--o{ LEARNING_ASSIGNMENTS : "assigned as"
    LEARNING_ASSIGNMENTS ||--o{ QUIZ_ATTEMPTS : has
    LEARNING_ASSIGNMENTS ||--o| CERTIFICATION_RECORDS : issues
    LEARNING_ASSIGNMENTS ||--o| OUTCOME_MEASUREMENTS : measures
    CERTIFICATIONS ||--o{ CERTIFICATION_RECORDS : "instance of"
    COURSES ||--o{ FAILURE_TO_COURSE_MAPPINGS : "targeted by"

    COURSES { uuid id PK }
    COURSE_VERSIONS { uuid id PK  uuid course_id FK  text lifecycle_state }
    QUIZ_QUESTIONS { uuid id PK  uuid course_version_id FK }
    LEARNING_ASSIGNMENTS { uuid id PK  uuid course_version_id FK  text status  text origin }
    QUIZ_ATTEMPTS { uuid id PK  uuid learning_assignment_id FK }
    CERTIFICATIONS { uuid id PK }
    CERTIFICATION_RECORDS { uuid id PK  uuid certification_id FK  uuid learning_assignment_id FK }
    OUTCOME_MEASUREMENTS { uuid id PK  uuid learning_assignment_id FK }
    FAILURE_TO_COURSE_MAPPINGS { uuid id PK  uuid course_id FK }
```
`Course`/`CourseVersion` follow the platform's Template/TemplateVersion immutable-after-publish
pattern. `LearningAssignment` is the central record, referencing an immutable `CourseVersion` and
fanning out to `QuizAttempt` (0..N), an optional `CertificationRecord` (issued on qualifying
completion), and an optional `OutcomeMeasurement` (only for failure-triggered assignments with a
linked re-inspection). `FailureToCourseMapping` is the Training Agent's read-only rule table.

## 5. State Diagram — LearningAssignment Lifecycle

```mermaid
stateDiagram-v2
    [*] --> ASSIGNED
    ASSIGNED --> IN_PROGRESS: content first viewed
    IN_PROGRESS --> COMPLETED: knowledge check passed (or no check required)
    IN_PROGRESS --> FAILED: max quiz attempts exhausted without passing
    ASSIGNED --> CANCELLED: role no longer applicable / user deactivated / superseded by reissue
    IN_PROGRESS --> CANCELLED: role no longer applicable / user deactivated / superseded by reissue
    COMPLETED --> VERIFIED: linked re-inspection passes (failure-triggered only)
    COMPLETED --> FAILED: linked re-inspection fails (no improvement)
    FAILED --> ASSIGNED: re-assigned (new LearningAssignment created; not a mutation of this row)
    COMPLETED --> [*]: terminal (no re-inspection linkage - most manual/role-based training)
    VERIFIED --> [*]
    CANCELLED --> [*]
```
Most manual and role-based assignments terminate at `COMPLETED` — there is no re-inspection to
verify against. Failure-triggered assignments continue: `COMPLETED` is provisional until the linked
re-inspection resolves it to `VERIFIED` (improvement measured) or `FAILED` (no improvement, which
triggers escalation to the Compliance Monitor Agent rather than an automatic identical re-assignment,
per functional-spec FR-LEARN-046). A `FAILED` assignment's "re-assignment" is always a **new**
`LearningAssignment` row (with a fresh `origin_ref` chain), never a resurrection of the failed row,
preserving an immutable audit history of every attempt.
