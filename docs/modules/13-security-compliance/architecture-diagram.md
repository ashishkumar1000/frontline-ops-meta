# Security, Privacy & Compliance — Architecture Diagrams

Module: `13-security-compliance`
Epic: EPIC-14

## 1. Component / container diagram

```mermaid
flowchart TB
    subgraph Clients
        WEB[Web Admin Console]
        MOBILE[Mobile App]
    end

    GW[API Gateway / BFF\nenforcement point: authN, tenant resolution, rate limit]

    subgraph ControlPlane["Security & Compliance Control-Plane Service"]
        POLICY[Policy Config API]
        AUDIT[Audit Aggregation API]
        KEYMGMT[Key-Rotation Orchestrator]
        DEVICE[Device Registry / Remote Wipe]
        DSR[DSR Orchestrator]
        INCIDENT[Incident & Vuln Tracking]
    end

    OPA[[OPA Policy Engine\nshared: authZ, tenant isolation, AI data controls]]

    subgraph Enforced["Distributed Enforcement Points (every other module)"]
        SVCPG[(Domain service Postgres + RLS)]
        OBJ[(Object storage: tenant-prefixed, KMS-encrypted)]
        SEARCH[(Search / vector store: tenant-filtered)]
        KAFKA[[Event backbone: tenant-partitioned]]
        AGENT[Agent Runtime context builder:\nredaction, allowlist, no cross-tenant retrieval]
    end

    KMS[(Managed KMS)]

    WEB --> GW
    MOBILE --> GW
    GW --> OPA
    GW --> SVCPG
    GW --> AGENT
    POLICY --> OPA
    POLICY -->|policy cache, 60s TTL| GW
    SVCPG -->|audit events| KAFKA --> AUDIT
    AGENT -->|audit events| KAFKA
    AUDIT --> AUDITSTORE[(audit_logs: hash-chained, append-only)]
    KEYMGMT --> KMS
    OBJ -.encrypt/decrypt.-> KMS
    SVCPG -.column encryption.-> KMS
    DEVICE --> MOBILE
    DSR --> SVCPG
    INCIDENT --> AUDITSTORE
    WEB --> POLICY & AUDIT & DSR & DEVICE & INCIDENT
```

**Narrative**: the control-plane service (top box) is a normal deployable that Enterprise Admins
and Security/Compliance officers interact with directly. The bottom box, "Distributed Enforcement
Points," is not a service — it is the same enforcement contract replicated inside every other
module's own service, object storage access, search/vector layer, event consumers, and the Agent
Runtime. Policy flows one way, down, from the control plane to a short-TTL cache each enforcement
point reads; audit events flow one way, up, from every enforcement point into the central
hash-chained store. OPA is the shared decision engine used both for ordinary authorization
(reused from Identity & Org) and for the tenant-isolation/AI-data-control checks this module adds.

## 2. Sequence diagram — end-to-end tenant-isolation enforcement

```mermaid
sequenceDiagram
    participant Client
    participant GW as API Gateway
    participant OPA as OPA
    participant SVC as Domain Service
    participant PG as Postgres (RLS)
    participant OBJ as Object Storage

    Client->>GW: Request (Bearer token, tenant A)
    GW->>GW: Resolve tenant_id from token only
    GW->>OPA: authorize(principal, action, resource)
    OPA-->>GW: ALLOW (tenant A, scope X)
    GW->>SVC: forward + verified tenant/scope
    SVC->>PG: SET app.tenant_id='A'; scoped query
    PG-->>SVC: rows (RLS-backstopped)
    SVC->>OBJ: request signed URL (key must start tenant/A/)
    OBJ-->>SVC: signed URL
    SVC-->>GW: 200 response
    GW-->>Client: 200 response
    SVC-)AUDIT: async audit event (hash-chained)
```

**Narrative**: this diagram is the literal walk-through of the §24.1 acceptance test — every hop
(gateway, OPA, service, Postgres RLS, object storage) independently proves tenant A scope before
acting, so a bug at any single layer does not, by itself, expose tenant B's data. The audit emit
is asynchronous to the response path (does not add latency to the read) but is still guaranteed
durable before the *originating* action is considered complete for security-critical writes
(not shown here, since this is a read path — see LLD §4.3 for the write-path guarantee).

## 3. Sequence diagram — DSR fulfillment across services

```mermaid
sequenceDiagram
    participant Admin
    participant DSR as DSR Orchestrator
    participant IDN as Identity Service
    participant WORK as Work Service
    participant EVID as Evidence Service

    Admin->>DSR: Submit DELETION request
    DSR->>DSR: status = IN_PROGRESS
    par
        DSR->>IDN: fulfill(subject, DELETION)
        IDN-->>DSR: FULFILLED (anonymized identity)
    and
        DSR->>WORK: fulfill(subject, DELETION)
        WORK-->>DSR: FULFILLED (actor anonymized, record kept)
    and
        DSR->>EVID: fulfill(subject, DELETION)
        EVID->>EVID: check RETENTION_HOLD
        EVID-->>DSR: EXCLUDED:LEGAL_HOLD or FULFILLED
    end
    DSR->>DSR: status = PARTIALLY_FULFILLED or FULFILLED
    DSR-->>Admin: outcome + excluded categories
```

**Narrative**: no single service can unilaterally "complete" a DSR — the orchestrator only marks
`FULFILLED` when every owning service reports back, and a `RETENTION_HOLD` on even one service's
data downgrades the outcome to `PARTIALLY_FULFILLED` with an explicit, auditable reason rather
than silently deleting around the hold or silently claiming full compliance.

## 4. ER diagram — owned entities

```mermaid
erDiagram
    SECURITY_POLICIES ||--o{ AUDIT_LOGS : "changes generate"
    DATA_SUBJECT_REQUESTS ||--o{ AUDIT_LOGS : "generates"
    DEVICE_REGISTRATIONS ||--o{ AUDIT_LOGS : "generates"
    SECURITY_INCIDENTS ||--o{ AUDIT_LOGS : "references"
    ENCRYPTION_KEY_METADATA ||--o{ SECURITY_INCIDENTS : "compromise triggers"

    SECURITY_POLICIES {
        uuid id PK
        uuid tenant_id
        int version
        boolean mfa_required
        boolean device_binding_required
        jsonb ai_redaction_rules
        text data_residency_region
    }
    AUDIT_LOGS {
        uuid id PK
        uuid tenant_id
        text action
        text resource_type
        text outcome
        bytea prev_hash
        bytea row_hash
        bigint chain_seq
    }
    DATA_SUBJECT_REQUESTS {
        uuid id PK
        uuid tenant_id
        uuid subject_ref
        text request_type
        text status
        jsonb excluded_categories
    }
    ENCRYPTION_KEY_METADATA {
        uuid id PK
        uuid tenant_id
        text key_scope
        text kms_key_ref
        text rotation_state
    }
    DEVICE_REGISTRATIONS {
        uuid id PK
        uuid tenant_id
        uuid user_id
        text status
        text attestation_result
    }
    SECURITY_INCIDENTS {
        uuid id PK
        text severity
        text category
        text status
        timestamptz patch_sla_due_at
    }
```

**Narrative**: `AUDIT_LOGS` is the hub every other owned entity connects to — every configuration
change, DSR action, device action, and incident is itself an audit event, which is what makes the
central audit store usable as compliance evidence for the module's own operation, not only for
other modules' business events.

## 5. State diagram — Data Lifecycle (source-spec §21.4)

```mermaid
stateDiagram-v2
    [*] --> CREATE
    CREATE --> ACTIVE : record committed / published
    ACTIVE --> ARCHIVED : retention-sweep job\n(active-period elapsed)
    ARCHIVED --> RETENTION_HOLD : legal hold / litigation /\nregulatory inquiry opened
    RETENTION_HOLD --> ARCHIVED : hold released by\nSecurity/Compliance officer
    ARCHIVED --> DELETE : retention period elapsed\nAND no active hold\nAND policy permits
    RETENTION_HOLD --> DELETE : hold released AND\nretention period already elapsed
    DELETE --> [*]

    note right of RETENTION_HOLD
        Optional detour. A record here cannot be
        deleted regardless of DSR deletion request
        or expired retention timer (FR-SEC-051).
    end note

    note right of DELETE
        DELETE/DESTROY only proceeds "where policy
        permits" — some record types (e.g. AuditLog
        within its own retention floor) never reach
        this state via a subject-initiated path.
    end note
```

**Narrative**: every tenant-owned record type with a lifecycle (`Submission`, `Evidence`,
`KnowledgeDocument`, `AgentRun`, and this module's own `AuditLog` under its distinct longer
retention) implements this exact four/five-state machine. `RETENTION_HOLD` is the deliberate
override point that makes the legal-hold-vs-DSR-deletion conflict (FSD §7) structurally
impossible to bypass: the state machine itself, not just application logic, blocks the
`DELETE`/`DESTROY` transition while a hold is active.
