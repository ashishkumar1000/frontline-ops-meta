# Gamification & Recognition — Architecture Diagrams

Module: `22-gamification`

## 1. Component Diagram

```mermaid
flowchart TD
    subgraph Upstream [Upstream Event Producers — not aware of Gamification]
        Learning[Learning\ntraining.completed]
        WorkOrch[Work Orchestration\ntask.completed / audit.submitted / streak signal]
        Engagement[Employee Engagement\nrecognition.event.created]
        IssuesCapa[Issues & CAPA\nfraud/invalidation signal]
    end

    Kafka[[Event Backbone]]

    subgraph Gam [Gamification Service]
        Eval[Badge Rule Evaluator]
        Ledger[Points Ledger Writer]
        Pathway[Pathway Progress Projector]
        Mat[Leaderboard Materializer]
    end

    subgraph Data [gamification schema]
        Awards[(badge_awards)]
        Points[(points_ledger_entries)]
        LB[(leaderboards)]
        PP[(pathway_progress)]
    end

    Web[Web Admin]
    Mobile[Mobile App]
    Notif[Notification Hub]

    Learning & WorkOrch & Engagement & IssuesCapa -- events --> Kafka
    Kafka --> Eval
    Kafka --> Ledger
    Kafka --> Pathway

    Eval --> Awards
    Ledger --> Points
    Pathway --> PP
    Eval -- badge.awarded --> Pathway
    Eval -- badge.awarded --> Notif

    Mat --> Points
    Mat --> Awards
    Mat --> LB

    Mobile --> Awards
    Mobile --> Points
    Mobile --> PP
    Mobile --> LB
    Web --> LB
```

**Narration**: This is an event-consumer architecture, not a request/response service — every arrow
into the Gamification Service from an upstream module is an asynchronous event, never a synchronous
call the upstream module waits on. The Badge Rule Evaluator, Points Ledger Writer, and Pathway
Progress Projector each independently consume the same event stream and write to their own tables;
the Leaderboard Materializer is the one component that runs on a schedule rather than reactively,
reading the already-durable ledger and award tables to produce the precomputed views mobile/web
clients actually query. No component here ever calls back into Learning, Work Orchestration, or
Employee Engagement to drive their behavior — the dependency arrow points one way.

## 2. Sequence Diagram — Badge Award From a Cross-Module Event

```mermaid
sequenceDiagram
    participant WO as Work Orchestration
    participant Kafka as Event Backbone
    participant Eval as Badge Rule Evaluator
    participant DB as gamification schema
    participant Notif as Notification Hub
    participant App as Frontline App

    WO->>Kafka: emit audit.submitted (violations_count = 0)
    Kafka->>Eval: deliver event (idempotency-checked)
    Eval->>DB: update rolling counter for (user, rule)
    alt threshold met (10th zero-violation audit)
        Eval->>DB: INSERT badge_award
        Eval->>DB: INSERT points_ledger_entry (bonus_points)
        Eval->>Notif: request badge-earned notification
        Notif-->>App: push / in-app "unseen award" flag
        App->>App: celebratory moment on next open
    else threshold not yet met
        Eval->>DB: counter updated only, no award
    end
```

**Narration**: The associate who triggers the tenth qualifying audit never interacts with
Gamification directly — the badge is a side effect of doing their normal job. The celebratory moment
is decoupled from the notification's delivery success: the award and points are already durably
committed before the notification is even attempted, so a lost push notification cannot cause a lost
badge, only a delayed celebration (surfaced instead via the "unseen award" flag on next app open).

## 3. Sequence Diagram — Points Reversal on Upstream Invalidation

```mermaid
sequenceDiagram
    participant Issues as Issues & CAPA
    participant Kafka as Event Backbone
    participant Ledger as Points Ledger Writer
    participant DB as gamification schema
    participant App as Frontline App

    Issues->>Kafka: emit issue.created (AUDIT_FRAUD, ref_event_id)
    Kafka->>Ledger: deliver invalidation event
    Ledger->>DB: locate original points_ledger_entry by source_event_id
    Ledger->>DB: INSERT reversal entry (negative amount, reversed_entry_id set)
    Ledger->>DB: check dependent badge_awards; revoke if condition no longer met
    Ledger->>App: (async, next sync) points history shows struck-through reversed entry
    Note over App: revocation UX is low-key/neutral —\nassociate is not assumed to be at fault
```

**Narration**: Reversal never mutates or deletes the original entry — it appends a compensating
transaction, preserving a complete, honest history (the platform's historical-truth principle applied
at this module's scale). A dependent badge is revoked only if, with the invalidated event excluded,
the rule's condition would no longer hold; the revocation UX is deliberately understated because the
associate receiving the badge is very often not the party responsible for the underlying invalidation.

## 4. ER Diagram — Owned Entities

```mermaid
erDiagram
    BADGES ||--o{ BADGE_RULES : "versioned by"
    BADGE_RULES ||--o{ BADGE_AWARDS : produces
    BADGE_AWARDS ||--o{ BADGE_REVOCATIONS : "may be reversed by"
    PATHWAYS ||--o{ PATHWAY_STEPS : has
    PATHWAYS ||--o{ PATHWAY_PROGRESS : "tracked per user"
    PATHWAYS ||--o| BADGES : "capstone is a"

    BADGES {
        uuid id PK
        string category
        string status
        int bonus_points
    }
    BADGE_RULES {
        uuid id PK
        uuid badge_id FK
        int rule_version
        jsonb condition
    }
    BADGE_AWARDS {
        uuid id PK
        uuid badge_id FK
        uuid user_id
        jsonb source_event_ids
        bool revoked
    }
    POINTS_LEDGER_ENTRIES {
        uuid id PK
        uuid user_id
        int amount
        string source_type
        uuid reversed_entry_id
    }
    LEADERBOARDS {
        uuid id PK
        string scope_type
        string time_window
        jsonb rows
        bool suppressed
    }
    PATHWAYS {
        uuid id PK
        string name
        string capstone_credential_name
    }
    PATHWAY_STEPS {
        uuid id PK
        uuid pathway_id FK
        int sequence
        string step_type
    }
    PATHWAY_PROGRESS {
        uuid id PK
        uuid pathway_id FK
        uuid user_id
        string status
        int current_step_index
    }
```

**Narration**: `POINTS_LEDGER_ENTRIES` and `LEADERBOARDS` are deliberately not drawn with a rigid FK
into `BADGE_AWARDS`/`PATHWAYS` — the ledger is an independent, append-only record that merely
*references* a badge award when relevant (`related_badge_award_id`), and leaderboards are a
*computed* projection over the ledger and award tables, refreshed by a batch job rather than kept
live-consistent by a database constraint. This separation is what makes the ledger safely reversible
without needing to touch or recompute badge/pathway state synchronously.

## 5. State Diagram — PathwayProgress Lifecycle

```mermaid
stateDiagram-v2
    [*] --> NOT_STARTED
    NOT_STARTED --> IN_PROGRESS : first step's course/badge completed
    IN_PROGRESS --> IN_PROGRESS : next sequential step completed (current_step_index++)
    IN_PROGRESS --> COMPLETED : final step completed

    note right of NOT_STARTED
        Created implicitly the first time
        a user is in scope for the pathway
        (e.g. role-based assignment) —
        no explicit "enroll" action needed
    end note

    note right of IN_PROGRESS
        Steps complete strictly in order
        (sequential = true, MVP-only mode).
        Each completion is driven by a
        consumed training.completed or
        badge.awarded event, never a
        direct user action on this module.
    end note

    note right of COMPLETED
        If capstone_credential_name is set,
        completion triggers award of the
        capstone Badge through the same
        Badge Rule Evaluator pipeline —
        not a special-cased code path.
    end note

    COMPLETED --> [*]

    IN_PROGRESS --> IN_PROGRESS : frozen=true on user.deactivated\n(progress preserved, not advanced)
```

**Narration**: `PathwayProgress` has no failure/reset terminal state — a pathway is not something a
learner can "fail," only pause (frozen on deactivation) or complete. Every transition is driven purely
by events this module already consumes for badges and points (`training.completed`, `badge.awarded`),
so a Pathway is, structurally, just a saved sequence of badge/course completions with a progress bar
drawn over it — consistent with this module's positioning as a lightweight visualization and
motivation layer rather than a new execution engine (`new-modules-addendum.md` row 22).
