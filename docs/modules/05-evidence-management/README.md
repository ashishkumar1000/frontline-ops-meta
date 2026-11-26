# Module 05 — Evidence Management

> Stores and protects every piece of proof that frontline work actually happened — photos, videos, audio, documents, signatures, GPS points, barcode scans, even AI verification results. Every item gets a tamper-evident hash, a chain-of-custody record (who captured what, when, where, on which device), tenant-configured retention, and legal-hold protection. Frontline workers create the evidence; managers, auditors, compliance teams, and the Visual AI module consume it.

| | |
|---|---|
| **Module code** | None assigned in the spec (identified by folder `05-evidence-management`) |
| **Owning service** | Evidence Service (metadata + signed-access control plane; media bytes live in S3-compatible object storage) |
| **Epics** | EPIC-06 |

## In plain terms
- Accept ten evidence types — photo, video, audio, document, signature, GPS, timestamp, device metadata, barcode/QR scan, and AI verification result — each with type-specific handling.
- Prove nothing was altered: a SHA-256 hash is computed at upload and re-checked at rest; any mismatch fails the upload or triggers a security alert, never a silent acceptance.
- Keep the audit trail straight: chain-of-custody fields are write-once, corrections are layered on top as audit entries, and every access to evidence bytes is recorded.
- Protect what matters legally: place legal holds that block deletion regardless of retention policy, and export bulk evidence packages with full manifests for audits or litigation.
- Respect privacy: sensitive content (faces, license plates, PII) gets a redacted copy as the default view, while the untouched original stays preserved and permission-gated.
- Manage storage cost automatically: identical files are de-duplicated by hash, evidence moves to cold archive after ~90 days of inactivity, and deletion only happens when retention lapses with no legal hold or open issue attached.

## Key concepts
| Concept | What it means |
|---|---|
| Evidence | The first-class record of one captured proof — metadata, lifecycle state, storage location, and hash; never just "an attachment" on a submission. |
| content_hash | The SHA-256 of the exact bytes, set once at verified upload and immutable; the basis of tamper detection and de-duplication. |
| Chain of custody | The write-once capture facts: who, when (device + server time), where (GPS), which device/app, and which form field the evidence answers. |
| evidence_links | The polymorphic link to exactly one primary parent (Submission, WorkItem, or Issue), plus optional secondary cross-references (e.g. an issue citing the same photo). |
| Evidence lifecycle | `PENDING_UPLOAD → UPLOADED → PROCESSING → AVAILABLE → [LEGAL_HOLD] → ARCHIVED → DELETED`, with a terminal `FAILED` state for hash mismatches or malware. |
| Retention policy | Per-tenant, per-evidence-class rules for how long evidence stays hot, then archived, before it becomes deletable. |
| Legal hold | A compliance freeze on selected evidence that suspends all deletion — releasing it only makes it eligible for the next retention sweep. |
| Signed URL | The only way bytes move: short-lived, single-object, single-operation URLs issued by the Evidence Service, so media never flows through app servers. |

## How it connects to the rest
- Receives uploads from the Mobile & Offline module (04), which owns on-device capture and the resumable upload queue; Evidence owns integrity and lifecycle once bytes land.
- Serves the Forms module (03), which marks fields as evidence-bearing and stores only evidence references; missing mandatory evidence fails the submission as a standard validation error.
- Feeds Visual AI (09), which consumes `evidence.processed` events and writes its AI verification result back as a new linked Evidence record; also feeds Issues & CAPA (06), Analytics (11), and Agent Runtime (10) tools.
- Depends on Security & Compliance (13) for per-tenant encryption keys (KMS), platform encryption standards, and the shared tamper-evident AuditLog it writes into.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-EVD-001–106: all ten evidence types and their capture contracts, chain-of-custody rules, the capture-to-storage lifecycle, hashing and integrity, linking, retention, legal hold and export, redaction, and the signed-URL access model. |
| [high-level-design.md](./high-level-design.md) | The control-plane/bytes-plane split, the event-driven processing pipeline (hash → thumbnail → transcode → malware scan → redaction detection), scalability via resumable multipart upload and CDN, multi-tenancy (per-tenant KMS keys), and failure modes. |
| [low-level-design.md](./low-level-design.md) | Full Postgres schema (`evidence`, `evidence_links`, processing jobs, retention policies, legal holds), the REST endpoint contracts with example payloads, the retention-sweep and orphan-GC pseudocode, and background jobs. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: the control-plane component diagram, the resumable upload with hash-verification sequence, the processing-pipeline sequence through to Visual AI, the ER diagram, and the full evidence lifecycle state machine. |
