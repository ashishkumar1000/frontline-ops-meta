# Communications & Knowledge — Architecture Diagrams

Module: `07-communications-knowledge`

## 1. Component / Container Diagram

```mermaid
flowchart TB
    subgraph Clients
        MOBILE[Mobile App]
        WEBADMIN[Web Admin / Control Tower]
        COPILOT[Operations Copilot Agent]
        COMMAGENT[Communication Agent]
    end

    BFF[API Gateway / BFF]

    subgraph CommsKnowledge[Comms & Knowledge Bounded Context]
        subgraph CS[CommunicationsService]
            AUTHOR[Authoring API]
            AUDRES[Audience Resolver]
            SCHED[Publish/Expiry Scheduler]
            DELIVERY[Delivery Orchestrator]
            ACKTRACK[Ack / Read-Receipt Tracker]
        end
        subgraph KS[KnowledgeService]
            DOCAPI[Document Authoring API]
            INDEXER[Indexing Pipeline]
            SEARCHAPI[Search API]
            RAGAPI[Ask-a-Question RAG API]
        end
    end

    PG[(Postgres: comms_knowledge)]
    OS[(OpenSearch)]
    VEC[(pgvector)]
    KAFKA[[Event Backbone]]
    NOTIFHUB[Notification Hub]
    REDIS[(Redis)]
    IDENTITY[Identity & Organization]

    MOBILE --> BFF --> AUTHOR
    WEBADMIN --> BFF
    COMMAGENT --> AUTHOR
    COPILOT --> RAGAPI
    BFF --> DOCAPI
    BFF --> SEARCHAPI
    BFF --> RAGAPI
    BFF --> ACKTRACK

    AUTHOR --> AUDRES --> IDENTITY
    AUDRES <--> REDIS
    AUTHOR --> SCHED --> DELIVERY --> NOTIFHUB
    AUTHOR --> PG
    ACKTRACK --> PG
    CS --> KAFKA

    DOCAPI --> PG
    DOCAPI --> KAFKA --> INDEXER
    INDEXER --> OS
    INDEXER --> VEC
    SEARCHAPI --> OS
    RAGAPI --> OS
    RAGAPI --> VEC
    RAGAPI <--> REDIS
```

**Narrative**: A single bounded context is split into two deployables — `CommunicationsService`
(write-heavy, fan-out-heavy) and `KnowledgeService` (read/search-heavy) — sharing an
`AudienceResolver` pattern against Identity & Organization and a Redis cache tier, but each with its
own scaling and failure profile. Announcements flow through a Temporal-backed scheduler into
Notification Hub fan-out; knowledge documents flow through Kafka into an indexing pipeline feeding
OpenSearch and pgvector. Both external agent surfaces — the Communication Agent (writes drafts) and
the Operations Copilot (reads via RAG) — go through the same governed APIs as human users, with no
privileged bypass path.

## 2. Sequence Diagram — Announcement Fan-Out

```mermaid
sequenceDiagram
    participant Author as Admin
    participant API as CommunicationsService
    participant Resolver as AudienceResolver
    participant Deliv as Delivery Orchestrator
    participant Notif as Notification Hub
    participant Mobile as Mobile Client

    Author->>API: publish(announcement_id)
    API->>Resolver: resolve_audience()
    Resolver-->>API: recipient set (deduplicated)
    API->>API: snapshot announcement_targets, status=PUBLISHED
    API->>Deliv: enqueue batched fan-out
    loop batches
        Deliv->>Notif: send_notification(batch)
        Notif-->>Deliv: delivery status
    end
    Notif-->>Mobile: push
    Mobile->>API: ack(announcement_id)
    API->>API: idempotent upsert announcement_acks
```

**Narrative**: Publish snapshots the resolved, deduplicated recipient set once, then fan-out proceeds
in bounded batches against the Notification Hub, keeping the write path independent of per-recipient
synchronous work. Acknowledgement is idempotent on `client_uuid`, so offline-queued or retried acks
never double-record — required for correctness at 1M+ device scale and for mobile offline replay.

## 3. Sequence Diagram — Authorized Knowledge Search / RAG

```mermaid
sequenceDiagram
    participant User as Frontline User / Copilot
    participant API as KnowledgeService
    participant Resolver as AudienceResolver
    participant OS as OpenSearch
    participant Vec as pgvector
    participant Model as Model Gateway

    User->>API: ask("What temp triggers corrective action?")
    API->>Resolver: resolve caller's authorized scope/role/attributes
    Resolver-->>API: authorized document_id set
    API->>OS: lexical query filtered to authorized_ids + effective window
    OS-->>API: candidate authorized chunks
    API->>Vec: semantic re-rank within candidate set only
    Vec-->>API: top-k chunks + citations
    alt no authorized passages
        API-->>User: {answer: null, reason: NO_AUTHORIZED_SOURCE_FOUND}
    else
        API->>Model: synthesize grounded answer
        Model-->>API: answer text
        API-->>User: {answer, citations[]}
    end
```

**Narrative**: Authorization is resolved and applied as a hard filter *before* any semantic search
or generation happens — the system never runs an unfiltered similarity search and post-filters. This
is what lets the same endpoint safely serve both a human's direct search and the Operations
Copilot's `search_knowledge()` tool call: the authorization boundary is identical for both callers.

## 4. ER Diagram — Module-Owned Entities

```mermaid
erDiagram
    ANNOUNCEMENTS ||--o{ ANNOUNCEMENT_AUDIENCE_RULES : targets
    ANNOUNCEMENTS ||--o{ ANNOUNCEMENT_TARGETS : resolves_to
    ANNOUNCEMENTS ||--o{ ANNOUNCEMENT_TRANSLATIONS : has
    ANNOUNCEMENTS ||--o{ ANNOUNCEMENT_DELIVERIES : delivered_as
    ANNOUNCEMENTS ||--o{ ANNOUNCEMENT_ACKS : acknowledged_via
    ANNOUNCEMENTS ||--o{ ANNOUNCEMENT_ACK_WORKITEMS : escalation_tracked_via
    KNOWLEDGE_DOCUMENTS ||--o{ KNOWLEDGE_DOCUMENT_VERSIONS : versioned_as
    KNOWLEDGE_DOCUMENTS ||--o{ KNOWLEDGE_DOCUMENT_AUDIENCES : scoped_by
    KNOWLEDGE_DOCUMENT_VERSIONS ||--o{ KNOWLEDGE_EMBEDDINGS : chunked_into

    ANNOUNCEMENTS {
        uuid id PK
        string status
        string priority
        bool ack_required
    }
    ANNOUNCEMENT_AUDIENCE_RULES {
        uuid id PK
        uuid announcement_id FK
    }
    ANNOUNCEMENT_TARGETS {
        uuid id PK
        uuid announcement_id FK
        uuid user_id
        bool active
    }
    ANNOUNCEMENT_TRANSLATIONS {
        uuid id PK
        uuid announcement_id FK
        string language
        string status
    }
    ANNOUNCEMENT_DELIVERIES {
        uuid id PK
        uuid announcement_id FK
        string channel
        string status
    }
    ANNOUNCEMENT_ACKS {
        uuid id PK
        uuid announcement_id FK
        string receipt_type
    }
    ANNOUNCEMENT_ACK_WORKITEMS {
        uuid id PK
        string state
        int escalation_level
    }
    KNOWLEDGE_DOCUMENTS {
        uuid id PK
        uuid owner_user_id
        string category
    }
    KNOWLEDGE_DOCUMENT_VERSIONS {
        uuid id PK
        uuid document_id FK
        string language
        string status
        timestamptz effective_from
        timestamptz effective_to
    }
    KNOWLEDGE_DOCUMENT_AUDIENCES {
        uuid id PK
        uuid document_id FK
    }
    KNOWLEDGE_EMBEDDINGS {
        uuid id PK
        uuid version_id FK
        int chunk_index
    }
```

**Narrative**: `Announcement` and `KnowledgeDocument` are the two aggregate roots. Every
authorization-relevant fact (audience rules, per-recipient targets, per-recipient deliveries and
acks) hangs off `Announcement`; every content-lifecycle fact (versions, audience scoping, embedded
chunks) hangs off `KnowledgeDocument`. Neither aggregate stores `OrganizationNode`/`User`/`Role`
data directly — they reference IDs resolved live against Identity & Organization.

## 5. State Diagram — Announcement Lifecycle

```mermaid
stateDiagram-v2
    [*] --> DRAFT
    DRAFT --> SCHEDULED: publish_at set in future
    DRAFT --> PUBLISHED: publish immediately
    SCHEDULED --> PUBLISHED: publish_at reached
    SCHEDULED --> DRAFT: unschedule
    PUBLISHED --> EXPIRED: expires_at reached
    PUBLISHED --> RETRACTED: author retracts
    EXPIRED --> [*]
    RETRACTED --> [*]

    state PUBLISHED {
        [*] --> ACK_PENDING: ack_required = true
        ACK_PENDING --> ACK_OVERDUE: deadline passed, not acked
        ACK_OVERDUE --> ACK_ESCALATED: escalation ladder tier fires
        ACK_ESCALATED --> ACK_ESCALATED: next tier fires
        ACK_PENDING --> ACKNOWLEDGED: user acks
        ACK_OVERDUE --> ACKNOWLEDGED: user acks late
        ACK_ESCALATED --> ACKNOWLEDGED: user acks late
        ACK_PENDING --> ACK_CANCELLED: user deactivated
        ACK_OVERDUE --> ACK_CANCELLED: user deactivated
        ACK_ESCALATED --> ACK_CANCELLED: user deactivated
    }
```

**Narrative**: The outer lifecycle (`DRAFT → SCHEDULED → PUBLISHED → EXPIRED`/`RETRACTED`) governs
visibility and delivery eligibility. Nested within `PUBLISHED`, each targeted recipient has an
independent acknowledgement sub-state machine — this is what allows one announcement to be
simultaneously `ACKNOWLEDGED` for some recipients, `ACK_ESCALATED` for others, and `ACK_CANCELLED`
for a recipient who was deactivated mid-campaign, without any global lifecycle contradiction.
