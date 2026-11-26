# Frontline Mobile & Offline — Architecture Diagrams

Module corresponds to **EPIC-05 (Mobile Offline)**. Focused diagram set for this module only;
narrative detail lives in `functional-spec.md`, `high-level-design.md`, and `low-level-design.md`.

## 1. Component / Container Diagram

```mermaid
flowchart TB
    subgraph DEVICE["Mobile Device (iOS / Android)"]
        UI["UI Layer<br/>Today / Next / Manager / Capture"]
        LDB[("WatermelonDB<br/>local cache")]
        ENGINE["Sync Engine<br/>queue, retry/backoff, conflict detection"]
        SEC[("Secure Storage<br/>tokens, keys")]
        subgraph NATIVE["Native Bridge (Kotlin/Swift)"]
            CAM["Camera Pipeline"]
            SCAN["Barcode/QR"]
            BG["Background Tasks"]
            BIO["Biometrics"]
        end
        STAGE[("Encrypted Media Staging")]
    end
    UI --> LDB
    UI --> CAM & SCAN
    CAM --> STAGE
    LDB <--> ENGINE
    STAGE <--> ENGINE
    ENGINE --> BG
    SEC -.-> ENGINE
    ENGINE <-->|HTTPS| GW["API Gateway / BFF"]
    GW --> SYNCSVC["Sync Service"]
    SYNCSVC --> WORK["Work Service"]
    SYNCSVC --> FORM["Form Service"]
    SYNCSVC --> EVID["Evidence Service"]
    SYNCSVC --> AUTH["Authorization Service"]
    EVID --> OBJ[("Object Storage")]
    SYNCSVC --> PG[("Postgres")]
    NOTIF["Notification Hub"] <-->|push / token| DEVICE
```
The device holds only a bounded, disposable cache (WatermelonDB) plus a protected outbound queue;
the Sync Service is the single mobile-facing façade over Work/Form/Evidence/Authorization Services,
so no other backend service talks to a device directly except the Notification Hub's push channel.

## 2. Sequence — New Device Bootstrap

```mermaid
sequenceDiagram
    participant App as Mobile App
    participant Auth as Identity/Auth
    participant Sync as Sync Service
    App->>Auth: Login (OIDC)
    Auth-->>App: tokens
    App->>Sync: POST /sync (sync_token=null)
    Sync-->>App: permissions, locations, templates, work_items (page 1) + next_sync_token
    App->>App: write local DB transactionally
    App->>Sync: POST /sync (sync_token=page-1 cursor)
    Sync-->>App: page 2 + final next_sync_token
    App->>App: Today renders fully offline-capable
```
Bootstrap is the one flow that requires connectivity; it is paginated and checkpointed so an
interruption resumes from the last acknowledged cursor rather than restarting.

## 3. Sequence — Offline Capture to Sync (with Conflict)

```mermaid
sequenceDiagram
    participant User
    participant App as Mobile App (offline)
    participant Sync as Sync Service
    User->>App: Capture evidence, submit WorkItem
    App->>App: write evidence + submission + sync_queue atomically
    App->>App: connectivity restored → background sync fires
    App->>Sync: POST /sync (batched changes)
    Sync-->>App: submission accepted; work_item_status conflict (409-class)
    App->>App: mark submission SYNCED; mark WorkItem CONFLICT
    User->>App: resolve conflict (per-field keep mine/use server's)
    App->>Sync: POST /sync (resolution)
    Sync-->>App: accepted
```
Submission acceptance and WorkItem-field conflict are independent outcomes within one batch
response — a conflict on a mutable task field never blocks or rolls back an already-valid,
immutable submission.

## 4. Sequence — Resumable Media Upload with Interruption

```mermaid
sequenceDiagram
    participant App as Mobile App
    participant Evid as Evidence Service
    App->>Evid: init upload (size, checksum, chunk size)
    Evid-->>App: upload_session_id
    loop chunks 0..N
        App->>Evid: PUT chunk i
    end
    Note over App,Evid: connection drops mid-upload
    App->>Evid: GET status (on reconnect)
    Evid-->>App: last_acked_chunk_index
    loop remaining chunks
        App->>Evid: PUT chunk i
    end
    App->>Evid: complete (checksum)
    Evid-->>App: server_evidence_id, UPLOADED
```
Large evidence media never transits the same request path as JSON sync deltas, and never restarts
from byte zero after an interruption — only the unacknowledged remainder is re-sent.

## 5. Local On-Device ER Diagram

```mermaid
erDiagram
    USERS ||--o{ PERMISSIONS : has
    LOCATIONS ||--o{ PERMISSIONS : "scoped to"
    LOCATIONS ||--o{ WORK_ITEMS : hosts
    TEMPLATE_VERSIONS ||--o{ WORK_ITEMS : renders
    TEMPLATE_VERSIONS ||--o{ SUBMISSIONS : "referenced by"
    WORK_ITEMS ||--o{ SUBMISSIONS : produces
    SUBMISSIONS ||--o{ EVIDENCE : attaches
    SUBMISSIONS ||--o{ SYNC_QUEUE : queues
    WORK_ITEMS ||--o{ SYNC_QUEUE : queues
    EVIDENCE ||--o{ SYNC_QUEUE : queues

    USERS { string id PK }
    PERMISSIONS { string id PK  number expires_at }
    LOCATIONS { string id PK  boolean is_assigned_to_user }
    TEMPLATE_VERSIONS { string id PK  boolean is_pinned }
    WORK_ITEMS { string id PK  string status  number version }
    SUBMISSIONS { string id PK  string client_uuid UK  string status }
    EVIDENCE { string id PK  string upload_status  number last_acked_chunk_index }
    SYNC_QUEUE { string id PK  string state  number client_sequence }
```
`template_versions` and unsynced `submissions`/`evidence`/`sync_queue` rows are the only data
treated as protected/pinned; everything else is a disposable, TTL'd cache subset of server state.

## 6. State Diagram — Sync / Connectivity States

```mermaid
stateDiagram-v2
    [*] --> OFFLINE
    OFFLINE --> SYNCING: connectivity restored / periodic trigger
    SYNCING --> SYNCED: all queued items accepted, no conflicts
    SYNCING --> CONFLICT: server returns conflicts[] for mutable entities
    SYNCING --> ERROR: 4xx (non-conflict, non-transient) or exhausted retries
    SYNCING --> OFFLINE: connectivity lost mid-cycle (unacked items requeued)
    SYNCED --> OFFLINE: connectivity lost
    SYNCED --> SYNCING: new local change queued + connectivity present
    CONFLICT --> SYNCING: user resolves conflict, resolution re-queued
    ERROR --> SYNCING: user retries / next backoff attempt
    ERROR --> OFFLINE: connectivity lost while in error backoff
    SYNCED --> [*]: logout (no queued items)
```
`OFFLINE` is a normal, expected steady state, not a fault — matching the product principle that
offline is a supported operating mode, not an error path. `CONFLICT` and `ERROR` are the only
states requiring explicit user or operator attention; every other transition is automatic.
