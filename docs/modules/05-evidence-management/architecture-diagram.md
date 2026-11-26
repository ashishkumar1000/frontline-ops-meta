# Evidence Management — Architecture Diagrams

Module corresponds to **EPIC-06 (Evidence)**. Focused diagram set only; narrative detail lives in
`functional-spec.md`, `high-level-design.md`, and `low-level-design.md`.

## 1. Component / Container Diagram

```mermaid
flowchart TD
    subgraph Client["Capture Clients"]
        MOBILE["Mobile App (React Native)"]
        WEB["Web Admin"]
    end

    GW["API Gateway / BFF"]

    subgraph EvidenceSvc["Evidence Service (Kotlin/Spring Boot)"]
        API["Evidence API"]
        SIGN["Signed URL Issuer"]
        RETSWEEP["Retention Sweeper"]
        ORPHANGC["Orphan Evidence GC"]
    end

    subgraph Storage["S3-Compatible Object Storage"]
        HOT["Hot tier (per-tenant prefix, KMS)"]
        COLD["Cold/archive tier"]
    end

    subgraph Pipeline["Processing Pipeline Workers"]
        HASH["Hash Verify"]
        THUMB["Thumbnail / Waveform"]
        TRANSCODE["Video Transcode"]
        AVSCAN["Malware Scan"]
        REDACT["Redaction Detector"]
    end

    PG[("Postgres: evidence, evidence_links,\nevidence_processing_jobs,\nretention_policies, legal_holds")]
    KAFKA[["Event Backbone (Kafka)"]]
    CDN["CDN (thumbnails/proxies)"]

    VISUALAI["Visual AI"]
    ISSUES["Issues / CAPA"]
    ANALYTICS["Analytics (CDC)"]
    AGENTS["Agent Runtime"]

    MOBILE --> GW
    WEB --> GW
    GW --> API
    API --> SIGN
    API --> PG
    SIGN -. signed URLs .-> MOBILE
    MOBILE -- "direct PUT/GET bytes" --> HOT
    HOT -- CDN-fronted reads --> CDN --> MOBILE
    API -- "evidence.uploaded" --> KAFKA --> Pipeline
    HASH --> THUMB --> AVSCAN --> REDACT
    Pipeline -- derived objects --> HOT
    Pipeline -- "evidence.processed" --> KAFKA
    KAFKA --> VISUALAI
    KAFKA --> ISSUES
    KAFKA --> ANALYTICS
    KAFKA --> AGENTS
    RETSWEEP --> PG
    RETSWEEP --> HOT
    RETSWEEP --> COLD
    ORPHANGC --> PG
```

The Evidence Service is a thin metadata + signed-access control plane; capture clients exchange
bytes directly with object storage, keeping the service's compute footprint independent of media
volume. The processing pipeline is a set of independently scalable, event-driven workers reacting
to `evidence.uploaded`, writing derived objects back to storage and publishing `evidence.processed`
for downstream consumers (Visual AI, Issues/CAPA, Analytics, Agent Runtime) to react to
asynchronously. Two background jobs — the retention sweeper and the orphan-evidence garbage
collector — operate directly against Postgres and object storage on a schedule, independent of the
request path.

## 2. Sequence Diagram — Upload (Resumable Multipart with Hash Verification)

```mermaid
sequenceDiagram
    participant M as Mobile Client
    participant ES as Evidence Service
    participant OS as Object Storage

    M->>ES: initiateUpload (type, size, client_hash)
    ES->>OS: create multipart upload
    ES-->>M: evidence_id, signed part URLs
    loop resumable parts
        M->>OS: PUT part (resumes on drop/expiry)
    end
    M->>ES: completeUpload (parts, client_computed_hash)
    ES->>OS: complete multipart + fetch bytes
    ES->>ES: compute SHA-256, compare to declared hash
    alt match
        ES-->>M: 200 UPLOADED
        ES->>ES: emit evidence.uploaded
    else mismatch
        ES-->>M: 409 EVIDENCE_HASH_MISMATCH (status=FAILED)
    end
```

Hash verification happens server-side immediately at upload-complete, not deferred to the
processing pipeline, so tamper/corruption during transit is caught before the record is ever
considered `UPLOADED`. Resumability means a dropped connection mid-upload never forces a
from-scratch re-capture — a core requirement given intermittent frontline connectivity.

## 3. Sequence Diagram — Processing Pipeline

```mermaid
sequenceDiagram
    participant ES as Evidence Service
    participant EB as Kafka
    participant W as Processing Worker
    participant VA as Visual AI

    ES->>EB: evidence.uploaded
    EB->>W: deliver
    W->>W: malware scan
    alt malware found
        W->>ES: status=FAILED (MALWARE_DETECTED), quarantine
    else clean
        W->>W: thumbnail / transcode / waveform
        W->>W: redaction-candidate detection
        W->>ES: status=AVAILABLE
        ES->>EB: evidence.processed
        EB->>VA: deliver (PHOTO/VIDEO types)
        VA->>ES: register AI_RESULT evidence (source_evidence_id link)
    end
```

The pipeline is a strict linear gate per object (scan → derive → flag) before an evidence item
becomes `AVAILABLE`; a malware hit is a hard stop that never reaches downstream consumers. Visual
AI consumes `evidence.processed` asynchronously and writes its result back as its own linked
`Evidence` row rather than mutating the source media record.

## 4. ER Diagram — Owned Entities

```mermaid
erDiagram
    EVIDENCE ||--o{ EVIDENCE_LINKS : "linked via"
    EVIDENCE ||--o{ EVIDENCE_PROCESSING_JOBS : "processed via"
    EVIDENCE ||--o{ LEGAL_HOLD_EVIDENCE : "held via"
    EVIDENCE }o--|| RETENTION_POLICIES : "governed by"
    EVIDENCE ||--o| EVIDENCE : "AI result -> source media"
    LEGAL_HOLDS ||--o{ LEGAL_HOLD_EVIDENCE : "covers"

    EVIDENCE {
        uuid id PK
        uuid tenant_id
        text evidence_type
        text status
        text content_hash
        uuid captured_by
        timestamptz captured_at
        boolean legal_hold
    }
    EVIDENCE_LINKS {
        uuid evidence_id FK
        text parent_type
        uuid parent_id
        text field_ref
    }
    EVIDENCE_PROCESSING_JOBS {
        uuid evidence_id FK
        text job_type
        text status
    }
    RETENTION_POLICIES {
        uuid id PK
        text evidence_class
        int active_retention_days
    }
    LEGAL_HOLDS {
        uuid id PK
        text hold_reason
        text status
    }
    LEGAL_HOLD_EVIDENCE {
        uuid legal_hold_id FK
        uuid evidence_id FK
    }
```

`Evidence` is the aggregate root; `evidence_links` is deliberately polymorphic (rather than three
separate FK columns) so a single evidence item can be referenced by a Submission and later
cross-referenced by an Issue without duplication. `retention_policies` and `legal_holds` are
tenant-level configuration/state that the retention sweeper joins against, not per-evidence data.

## 5. State Diagram — Evidence Lifecycle

```mermaid
stateDiagram-v2
    [*] --> PENDING_UPLOAD: initiateUpload
    PENDING_UPLOAD --> UPLOADED: completeUpload (hash verified)
    PENDING_UPLOAD --> FAILED: hash mismatch / upload abandoned
    UPLOADED --> PROCESSING: evidence.uploaded consumed
    PROCESSING --> AVAILABLE: pipeline succeeds (scan clean, derivatives generated)
    PROCESSING --> FAILED: malware detected / transcode error / at-rest hash mismatch
    AVAILABLE --> LEGAL_HOLD: legal hold applied
    LEGAL_HOLD --> AVAILABLE: legal hold released
    AVAILABLE --> ARCHIVED: storage-tier transition (retention sweep, no hold)
    LEGAL_HOLD --> ARCHIVED: tier transition still applies while held (access unaffected)
    ARCHIVED --> LEGAL_HOLD: legal hold applied post-archive
    ARCHIVED --> DELETED: retention window elapsed AND no active legal hold AND no open referencing Issue
    FAILED --> [*]: terminal (no retry path re-uses same bytes/record)
    DELETED --> [*]: terminal, tombstoned then physically purged
```

This mirrors the platform Data Lifecycle in source-spec §21.4 (CREATE → ACTIVE → ARCHIVED →
RETENTION HOLD (optional) → DELETE/DESTROY), specialized for Evidence: `PENDING_UPLOAD`/`UPLOADED`/
`PROCESSING` are Evidence-specific pre-`ACTIVE` sub-states leading to `AVAILABLE` (the "ACTIVE"
equivalent), `LEGAL_HOLD` is the "RETENTION HOLD" state and can be entered from either `AVAILABLE`
or `ARCHIVED`, and deletion is only reachable when both the retention window has elapsed and no
hold or open referencing Issue blocks it (low-level-design.md §4.2).
