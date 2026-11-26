# Evidence Management — Functional Specification

Module corresponds to **EPIC-06 (Evidence)**. Source: source-spec.md §10.1 (Evidence types),
§17.1 (Evidence entity), §21 (Security, Privacy and Tenant Isolation — AI data controls and data
lifecycle relevant to media), Appendix E EPIC-06.

## 1. Purpose and Scope

### 1.1 Purpose
Evidence Management makes "a claim of completion and a verified completion are distinct states"
(Product Principle P4, source-spec §4) operationally real. It is the module that captures,
integrity-protects, stores, links, retains and controls access to every piece of proof that
frontline work actually happened as reported: photos, videos, audio, documents, signatures, GPS,
timestamps, device metadata, barcode/QR scans, and AI verification results (source-spec §10.1).
Evidence is a **first-class domain object** (`Evidence`, source-spec §17.1) — not an attachment
bag on a Submission — so it can be independently referenced, audited, held for legal/compliance
purposes, retained or purged under tenant-configured policy, and consumed by downstream services
(Visual AI, Issues/CAPA, Analytics, Agent Runtime) without those services owning storage or
integrity concerns.

The module is the system of record for **chain of custody**: who captured a piece of evidence,
when, where, on what device, under what app version, against what template field, and whether its
bytes have been altered since capture. It is deliberately split into a **metadata/control plane**
(Evidence Service, Postgres-backed) and a **bytes plane** (S3-compatible object storage, per
conventions.md), consistent with the Core Technical Components table (source-spec §16.3: "Evidence
Service | Metadata and signed access to media | Hashing, encryption, retention").

### 1.2 In Scope
- All ten evidence types from source-spec §10.1: Photo, Video, Audio, Document, Signature, GPS,
  Timestamp, Device metadata, Barcode/QR scan, AI verification result.
- The capture-to-storage lifecycle for each type: client capture → upload (direct-to-object-store
  via signed URL, resumable for large media) → server-side hash verification → processing
  (thumbnail/transcode) → availability → downstream consumption.
- Content-hash computation and integrity verification (tamper detection) for every evidence object.
- Chain-of-custody metadata: capturing user, capture timestamp (device + server), capture location
  (GPS where available), device identifier and OS/app version, template field the evidence was
  captured against.
- Evidence-to-parent linking: an `Evidence` record links to exactly one primary parent
  (`Submission`, `WorkItem`, or `Issue`) via a polymorphic association, and may additionally be
  cross-referenced by other objects (e.g., an Issue created from a failed Submission references the
  same Evidence).
- Per-tenant, per-evidence-class retention policy configuration and enforcement, including legal
  hold that suspends retention deletion.
- Redaction/PII handling workflow for evidence identified as containing sensitive content (faces,
  license plates, documents with PII) before AI processing or before export to lower-trust
  contexts.
- Signed-URL read/write access model: expiry, scoping, and re-issuance.
- Evidence export for audit response and legal hold (bulk, filtered, chain-of-custody-preserving
  export packages).
- UX states for evidence capture and sync on the mobile client, insofar as they are evidence-object
  states (upload-pending, upload-failed, thumbnail-generating, compression-in-progress, locked).

### 1.3 Out of Scope (owned by other modules, referenced here)
- The camera/microphone capture UI, local media file handling before upload, and the offline
  upload queue mechanics (retry scheduling, background task scheduling, local storage quota on
  device) — **Mobile/Offline module (EPIC-05)**. Evidence Management owns the server-side contract
  (signed upload URL issuance, upload-complete registration, hash verification) that the mobile
  client calls into.
- Definition of which form fields require which evidence types, field-level mandatory/conditional
  rules for evidence capture — **Forms module (EPIC-04)**. Evidence Management owns the evidence
  object itself once a field value references it, not the form schema.
- Interpretation of photo/video content (compliance scoring, bounding boxes, violation detection)
  — **Visual AI module (EPIC-10)**. Evidence Management stores the `AI verification result`
  evidence type as a structured record and emits `evidence.uploaded` for Visual AI to consume; it
  does not run inference.
- Issue/CAPA lifecycle and root-cause workflows that evidence supports — **Issues & CAPA module
  (EPIC-07)**. Evidence Management only owns the link and the bytes.
- Tenant-wide encryption key management, KMS provisioning, and platform-wide security policy —
  **Security & Compliance module (EPIC-14)**. Evidence Management consumes per-tenant KMS keys and
  enforces retention/legal-hold as configured but does not own key lifecycle.
- General audit log storage (`AuditLog` entity) — **Security & Compliance / Identity modules**.
  Evidence Management emits audit-relevant events but the tamper-evident audit trail itself is a
  shared platform service.

## 2. Personas Served

| Persona | Interaction |
|---|---|
| Frontline Associate | Captures evidence (photo/video/audio/signature/GPS/barcode) as part of completing a WorkItem/Submission; sees upload/sync state on-device |
| Auditor/Inspector | Captures evidence during audits/inspections, frequently at higher volume and higher evidentiary rigor (chain-of-custody expectations closer to legal-grade) |
| Store/Site Manager | Reviews evidence attached to submissions/issues during approval; requests additional evidence; views redacted vs. unredacted based on permission |
| Regional/District Manager, Executive | View evidence rolled up in exception/audit review, generally through Analytics/Control Tower thumbnails and signed links, not raw files |
| Auditor (external) / Legal / Compliance Admin | Places legal holds, runs bulk evidence export for audit or litigation response |
| AI Agent (Visual Inspection Agent, Compliance Monitor) | Consumes evidence bytes/metadata as an input to inference and pattern detection, via typed tool access that respects the same authorization and redaction rules as human access |
| Enterprise Admin | Configures per-tenant retention policy, evidence storage tiering, redaction rules, and evidence-class access policy |

## 3. Evidence Types and Capture Contract

Every `Evidence` record has an `evidence_type` and a type-specific `capture_payload` in addition to
the common chain-of-custody envelope (§4). All types share the same lifecycle (§5) and integrity
model (§6).

- **FR-EVD-001 — Photo.** JPEG/HEIC/PNG/WEBP accepted client-side; normalized to JPEG (with
  original retained per tenant policy, see FR-EVD-024) server-side for downstream Visual AI
  consistency. Client-captured EXIF (GPS, orientation, capture timestamp) is preserved as
  `capture_payload.exif` but the server independently records server-observed GPS/timestamp
  fields (§4) since EXIF is client-supplied and not trusted for authoritative facts.
- **FR-EVD-002 — Video.** MP4 (H.264/HEVC) accepted; max client-side capture length configurable
  per tenant (default 120s) to bound upload size on constrained networks. Server generates a
  poster-frame thumbnail and, per tenant configuration, a transcoded low-bitrate proxy for
  bandwidth-constrained playback (Control Tower / manager review).
  - **Resolved:** audio-track redaction (muting) is **not** supported in this version at the same
    granularity as visual (frame-region) redaction. Any video requiring audio redaction is routed to
    a manual human review queue where a reviewer either approves the video with audio intact or
    rejects/replaces the evidence; there is no automated audio-muting pipeline. This is firm because
    building a separate audio-redaction pipeline (waveform-segment detection/muting) is not
    justified by current requirements when the existing visual-redaction escalation path already
    covers the compliance need.
- **FR-EVD-003 — Audio.** M4A/AAC or WAV accepted; no thumbnail; a waveform preview is generated
  server-side for the review UI. Max duration configurable (default 300s).
- **FR-EVD-004 — Document.** PDF, DOCX, XLSX, common image formats accepted as a document
  evidence-type (distinct from Photo — used for uploaded existing files such as invoices,
  certificates, permits, rather than in-app camera captures). Virus/malware scan is mandatory
  before the object transitions out of `PROCESSING` (§5, §8 failure modes in HLD). Page-count and
  page-thumbnail extraction for PDFs.
- **FR-EVD-005 — Signature.** Captured as vector stroke data (array of point/pressure/timestamp
  samples) AND rendered to a PNG at capture time; both are stored — the vector data supports
  higher-fidelity legal defensibility and re-rendering, the PNG supports fast display. Signature
  evidence additionally records the `signer_identity` (may differ from `captured_by` when a
  customer/third party signs on an associate's device) and an attestation text hash (the exact
  text the signer was shown, e.g., a waiver or acknowledgement string) at capture time.
- **FR-EVD-006 — GPS.** A standalone evidence type in addition to being embedded in every other
  type's chain-of-custody envelope. Used when a form field explicitly requires an independent
  location-proof capture. Stores latitude, longitude, horizontal accuracy (meters), altitude
  (optional), and capture source (`GPS`, `NETWORK`, `PASSIVE`).
- **FR-EVD-007 — Timestamp.** A standalone evidence type for fields that require an attested
  point-in-time fact independent of any media (e.g., "time medication administered"). Always
  carries both `client_observed_at` and `server_received_at`; the delta between them is stored and
  surfaced (large deltas are a tamper/clock-skew signal, see §8 edge cases).
- **FR-EVD-008 — Device metadata.** Not typically a standalone evidence object a user "captures,"
  but a structured sub-record attached to every capture: device model, OS + version, app version +
  build number, device_id (per Mobile/Offline module device registration), network state at
  capture time (online/offline), and biometric/passcode-lock state if the device policy requires
  it. Exposed as `Evidence.device_metadata` on every record, and may also be requested as an
  explicit evidence item on high-rigor audit templates (`evidence_type = DEVICE_METADATA`) to
  create an independently reviewable record.
- **FR-EVD-009 — Barcode/QR scan.** Stores the decoded payload string, symbology
  (`CODE_128`, `QR`, `EAN_13`, etc.), and, when available, a captured photo of the scan target as
  corroborating evidence. Used both for evidence purposes and as a lookup key into reference data
  (Forms module concern) — Evidence Management only owns the scan-as-proof record.
- **FR-EVD-010 — AI verification result.** Structured JSON conforming to the Visual AI result
  contract (source-spec §12.2: `compliance_score`, `confidence`, `status`, `violations[]`).
  Always references the source Photo/Video `Evidence.id` it was computed from (`source_evidence_id`
  FK) and the `AgentRun`/model version that produced it. Never overwrites the source media
  evidence; stored as its own linked evidence record so both the raw capture and every AI
  interpretation of it (including re-runs after model upgrades) remain independently retrievable.

## 4. Chain-of-Custody Metadata (FR-EVD-020 to FR-EVD-023)

- **FR-EVD-020.** Every `Evidence` record must capture, at minimum, and immutably once set:
  `captured_by` (user id), `captured_at` (client-reported) and `received_at` (server-authoritative,
  per source-spec §19.4 "client clock cannot be trusted for authoritative ordering"), `device_id`,
  `app_version`, `capture_location` (GPS lat/lon/accuracy, nullable if unavailable — see FR-EVD-042),
  `capture_context` (which Submission/WorkItem/Issue and which template field/question it was
  captured against), and `content_hash` (§6).
- **FR-EVD-021.** Chain-of-custody fields are write-once at registration time (upload-complete
  call, §5) and are never mutated thereafter by any actor, including admins; corrections (e.g., a
  mis-tagged location) are represented as an `evidence_correction` audit entry layered on top, not
  as an edit to the original fact.
- **FR-EVD-022.** For evidence captured offline and synced later (Mobile/Offline module handoff),
  `captured_at` reflects true device capture time and `received_at` reflects sync-time server
  receipt; the gap is expected and must not be flagged as anomalous by itself, but is retained for
  audit/analytics (time-to-sync distribution).
- **FR-EVD-023.** Every access to evidence bytes (signed URL issuance) and every metadata mutation
  (link change, redaction applied, legal hold applied/released, retention override) is recorded to
  the shared `AuditLog` with actor, action, before/after, and timestamp, per source-spec §21.2/
  §17.1 AuditLog contract.

## 5. Capture-to-Storage Lifecycle

**FR-EVD-030.** The canonical Evidence lifecycle (detailed as a state machine in
`architecture-diagram.md`) is:

```
PENDING_UPLOAD -> UPLOADED -> PROCESSING -> AVAILABLE -> [LEGAL_HOLD] -> ARCHIVED -> DELETED
```

with a `FAILED` side-state reachable from `PENDING_UPLOAD`/`UPLOADED`/`PROCESSING`, consistent with
the platform Data Lifecycle (source-spec §21.4: CREATE → ACTIVE → ARCHIVED → RETENTION HOLD
(optional) → DELETE/DESTROY).

- **FR-EVD-031 — Initiate.** Client requests an upload (metadata pre-registration: evidence_type,
  parent reference, template field ref, declared content-length and mime-type, declared client
  hash if pre-computed). Server creates an `Evidence` row in `PENDING_UPLOAD`, allocates a
  per-tenant, per-evidence object storage key (`s3://<tenant-prefix>/evidence/<yyyy>/<mm>/<evidence_id>/<original_filename>`)
  and returns one or more signed PUT URLs (multipart for objects above a size threshold, default
  8 MB, per conventions.md resumable-upload guidance).
- **FR-EVD-032 — Upload.** Client uploads bytes directly to object storage (never proxied through
  application servers — see high-level-design.md §6 for rationale). For multipart, the client may
  pause/resume across network transitions (a core offline-mode requirement per source-spec §9.2).
- **FR-EVD-033 — Complete/Register.** Client calls upload-complete with the parts manifest (if
  multipart) and its locally computed content hash. Server verifies the object exists at the
  expected key with the expected size, computes/verifies the server-side hash (§6), and transitions
  `PENDING_UPLOAD → UPLOADED`. This call is idempotent on `(tenant_id, client_evidence_uuid)` — a
  retried complete-call after a dropped response returns the original result rather than erroring
  or reprocessing (Product Principle P9, source-spec §4).
- **FR-EVD-034 — Processing.** An async pipeline (triggered by `evidence.uploaded`) performs, per
  evidence type: thumbnail generation (photo/video/document), transcoding (video proxy, audio
  waveform), malware/virus scan (document, and any type accepting arbitrary file upload),
  redaction-candidate flagging (§7), and completes with `PROCESSING → AVAILABLE`. Malware detection
  or hash mismatch instead routes to `FAILED` with a reason code and blocks parent Submission
  approval where policy requires verified evidence.
- **FR-EVD-035 — Available.** Evidence is queryable, retrievable via signed URL, and eligible for
  downstream consumption (Visual AI, Analytics CDC, Agent Runtime tools) once `AVAILABLE`.
- **FR-EVD-036 — Legal Hold.** An `AVAILABLE` (or later-stage) evidence record can be placed under
  legal hold (§9), which blocks retention-driven deletion regardless of the evidence's retention
  policy expiry, without changing its primary lifecycle state for read/access purposes.
- **FR-EVD-037 — Archive/Delete.** Per tenant retention policy (§9), evidence past its active
  retention window (and with no legal hold) moves to cold storage tier (`ARCHIVED`, per
  high-level-design.md storage tiering) and eventually to `DELETED` (object storage bytes purged,
  metadata row tombstoned per source-spec §19.4 tombstone convention for a defined retention-log
  window before physical metadata deletion).

## 6. Hashing and Integrity Verification

- **FR-EVD-040.** Every evidence object's content hash is computed as SHA-256 over the final
  assembled byte stream (post multipart-reassembly, pre any server-side transformation such as
  thumbnailing). The hash is stored as `Evidence.content_hash` and is immutable once set.
- **FR-EVD-041.** If a client-supplied hash is present, the server-computed hash must match; a
  mismatch fails upload-complete with `EVIDENCE_HASH_MISMATCH` and transitions the record to
  `FAILED` rather than `UPLOADED` — this is the primary tamper/corruption detection mechanism for
  in-flight uploads.
- **FR-EVD-042.** Any subsequent access path that reads evidence bytes for re-verification
  (retention sweep touch, legal-hold export, periodic integrity audit job) recomputes the hash from
  the object-storage copy and compares to `Evidence.content_hash`; a mismatch (bytes altered at
  rest, e.g. by an out-of-band storage operation) raises a `SEV-1` integrity-violation alert and
  freezes the record's signed-URL issuance pending security review. This is distinct from
  hash-mismatch at upload time and reflects at-rest tamper detection.
- **FR-EVD-043 — Content-hash dedup.** When an incoming evidence upload's server-computed hash
  matches an existing `Evidence.content_hash` for the same tenant, the system does not duplicate
  object-storage bytes; it creates a new metadata row pointing at the existing object-storage key
  (reference-counted) while preserving independent chain-of-custody metadata per the new capture
  event. This commonly occurs when a user retries an upload after a dropped connection with the
  same source file (see §8 edge cases) or when the same signature/document is legitimately reused
  across a batch of submissions.

## 7. Linking Evidence to Parent Objects

- **FR-EVD-050.** Every `Evidence` record has exactly one primary parent via a polymorphic
  `evidence_links` association: `parent_type ∈ {SUBMISSION, WORK_ITEM, ISSUE}` + `parent_id`, plus
  the specific `field_ref` (template field/question id) it was captured against, when applicable.
  Standalone evidence (e.g., an ad hoc photo attached directly to an Issue, not via a form field)
  has a null `field_ref`.
- **FR-EVD-051.** Evidence may carry additional, non-primary cross-references (`evidence_links`
  rows with `link_role = SECONDARY`) — e.g., an `Issue` auto-created from a failed audit Submission
  references the same evidence the Submission already owns, without re-uploading or re-hashing
  bytes.
- **FR-EVD-052.** Evidence linkage is established at registration time (FR-EVD-031) using the
  client-supplied parent reference; if the parent (e.g., the draft Submission) does not yet exist
  server-side (common in offline capture, where evidence may sync before or after its parent
  Submission), the link is created in a `PENDING_PARENT` sub-state and resolved when the parent
  syncs (see FR-EVD-071 orphan handling).

## 8. Retention Policy Configuration

- **FR-EVD-060.** Retention is configured per tenant and per **evidence class** — a tenant-defined
  grouping (e.g., "Food Safety Photos," "Signatures," "Incident Video," "AI Results") that maps
  evidence records to a retention policy via `evidence_type` + optional `template_id`/`work_item_type`
  tags at capture time. Default system classes exist per evidence type if a tenant configures
  nothing.
- **FR-EVD-061.** A retention policy specifies: `active_retention_days` (hot/immediately-accessible
  window), `archive_retention_days` (cold-tier window after which deletion becomes eligible, `null`
  = retain indefinitely), and `legal_hold_override` (boolean, whether legal hold can suspend
  deletion for this class — default true).
- **FR-EVD-062.** Retention changes are not retroactive by default: a policy edit applies to
  evidence captured after the edit; tenants may explicitly request re-application to existing
  evidence via an admin action that is itself audit-logged given its destructive potential.
- **FR-EVD-063.** Retention enforcement never deletes evidence referenced by an `Evidence` record
  under active legal hold (§9), an open (non-`CLOSED`) `Issue`, or a `Submission`/`WorkItem` still
  within its own record-retention window, whichever is longer — retention is evaluated as the
  maximum of evidence-class policy and any referencing object's retention requirement.

## 9. Legal Hold and Audit Export

- **FR-EVD-070.** A Compliance Admin (or authorized role) may place a legal hold on evidence
  selected by filter (date range, location, evidence class, related Issue/Submission, or explicit
  ID list). Held evidence is flagged `legal_hold = true` with `hold_reason`, `placed_by`,
  `placed_at`, and an optional `case_reference`.
  Evidence under hold shows as **evidence-locked-under-legal-hold** in all UI surfaces: retention
  sweeps skip it entirely (§13 pseudocode), and any attempt to force-delete it via an admin API
  returns `409 Conflict` with code `EVIDENCE_UNDER_LEGAL_HOLD`.
- **FR-EVD-071.** Legal hold release requires the same or higher role than the one that placed it,
  is audit-logged, and does not retroactively delete evidence that has since passed its retention
  window — release simply makes it eligible for the next retention sweep.
- **FR-EVD-072 — Bulk export.** The system supports a bulk evidence export job (filtered the same
  way as legal hold) that produces a downloadable package: the original bytes (from cold/hot
  storage as applicable), a machine-readable manifest (chain-of-custody metadata, content hash,
  parent linkage, redaction status) per item, and a human-readable index. Exports are themselves
  audit-logged and, for large exports, delivered via a time-limited signed URL rather than inline
  API response.

## 10. Redaction and PII Handling

- **FR-EVD-080.** Evidence may be flagged (automatically by a redaction-candidate detector during
  processing, §5, or manually by a reviewer) as containing sensitive content: faces, license
  plates, government ID numbers, or other PII visible in photo/video/document evidence.
- **FR-EVD-081.** A flagged evidence item exposes two access forms: `raw` (original bytes,
  permission-gated to roles with an explicit "view unredacted evidence" grant) and `redacted` (a
  derived object — blurred/masked — generated by the processing pipeline and used as the default
  view for all other authorized roles, downstream Visual AI input where tenant policy requires it,
  and any bulk export unless the exporter holds the raw-access grant).
- **FR-EVD-082.** Per source-spec §21.3 (AI Data Controls), tenants may configure whether AI
  processing (Visual AI, agent tool consumption) receives raw or redacted evidence by default, and
  which evidence classes are excluded from AI processing entirely.
- **FR-EVD-083.** Redaction is never applied in place; the original raw evidence and its hash are
  always preserved (chain-of-custody and legal-defensibility requirement) — redaction always
  produces an additional derived, linked object.

## 11. Signed-URL Access Model

- **FR-EVD-090.** All evidence byte access (read and write) is via time-limited, single-object
  signed URLs issued by the Evidence Service; application servers and downstream services never
  proxy raw bytes (see high-level-design.md §6 for the throughput/security rationale).
- **FR-EVD-091.** Read signed URLs default to a 15-minute expiry for direct evidence detail views
  and a longer expiry (default 24 hours) for CDN-fronted thumbnails, both tenant-configurable
  within platform-enforced maximum bounds (30 days absolute maximum, to bound blast radius of a
  leaked URL).
- **FR-EVD-092.** Write signed URLs (upload PUT/multipart part URLs) expire at 60 minutes per part
  by default, reflecting expected mobile upload duration on constrained networks; a client whose
  URL expires mid-upload requests a fresh signed URL for the remaining parts (multipart uploads are
  resumable by design, not restarted from zero).
- **FR-EVD-093.** Every signed-URL issuance is scoped to exactly one object key and one operation
  (GET or PUT for a specific part), never a bucket-level or prefix-level grant, and is logged with
  requester identity and evidence id for audit purposes (not necessarily every GET call itself, to
  avoid audit-log volume explosion on thumbnail-heavy views, but at minimum rate-sampled/aggregated
  per source-spec §22 observability expectations — full per-access logging is mandatory for
  evidence under legal hold or flagged sensitive).

## 12. UX States

| State | Trigger / meaning | User-facing treatment |
|---|---|---|
| `upload-pending` | Evidence captured locally, queued for upload (online or offline) | Local thumbnail with a pending/spinner badge; visible in the Submission draft immediately |
| `upload-failed-retry` | Upload attempted and failed (network, storage-quota, validation) after exhausting automatic retry backoff | Explicit failed badge with manual "Retry" action; underlying local file preserved until success or explicit discard |
| `thumbnail-generating` | `UPLOADED`/early `PROCESSING`, byte upload succeeded but derived thumbnail not yet ready | Placeholder/blurred thumbnail with a subtle progress indicator; full evidence still viewable via original if requested |
| `large-file-compression-in-progress` | Video/large-photo compression or transcoding step of `PROCESSING` | Progress indicator distinct from generic "processing," since this step can take materially longer for video |
| `evidence-locked-under-legal-hold` | `legal_hold = true` | Lock iconography; delete/retention actions disabled; still viewable/exportable per normal permissions |
| (implicit) upload-quota-blocked | Tenant storage quota exceeded (§13) | Upload blocked client-side with a clear quota-exceeded message before bytes are attempted, where quota state is known/cached; server-side hard block otherwise |

## 13. Validation and Edge Cases

- **FR-EVD-100 — Duplicate evidence upload after retry.** A client that retries an upload-complete
  call (e.g., due to a dropped response after the server actually succeeded) must not create a
  duplicate `Evidence` row. Idempotency is enforced on `(tenant_id, client_evidence_uuid)`
  (FR-EVD-033); if the same source file is re-captured as a genuinely new evidence item, dedup
  instead applies at the content-hash level (FR-EVD-043), producing a new metadata row sharing
  storage bytes, not a rejected request.
- **FR-EVD-101 — Evidence orphaned when parent submission fails validation.** If a Submission fails
  server-side validation and is rejected (never persisted as a valid Submission), evidence already
  uploaded and linked to it via `client_evidence_uuid`/draft reference is not deleted immediately;
  it moves to an `ORPHAN_CANDIDATE` sub-state with a grace period (default 7 days, tenant
  configurable) during which the client can re-link it to a corrected resubmission without
  re-uploading bytes. Evidence still `ORPHAN_CANDIDATE` after the grace period is swept by the
  orphan-evidence garbage collector (low-level-design.md §7).
- **FR-EVD-102 — Tampered-hash detection.** See FR-EVD-041 (upload-time) and FR-EVD-042 (at-rest).
  Both cases produce a distinct, non-recoverable-by-retry error surfaced to the capturing user
  (upload-time) or a security alert (at-rest) — the system never silently accepts a hash mismatch.
- **FR-EVD-103 — Storage quota exceeded per tenant.** Each tenant has a configurable storage quota
  (bytes and/or object count) evaluated at upload-initiate time (FR-EVD-031). Exceeding it returns
  `EVIDENCE_QUOTA_EXCEEDED` and blocks new uploads for that tenant while allowing existing evidence
  to remain readable; Enterprise Admins are notified at configurable warning thresholds (e.g., 80%,
  95%) before hard block, and legal-hold/compliance-critical evidence classes can be configured to
  bypass soft quota warnings up to a hard ceiling.
- **FR-EVD-104 — GPS unavailable indoor capture.** When device GPS cannot resolve (common in
  large indoor facilities), `capture_location` is stored as `null` with a `location_source =
  UNAVAILABLE` flag rather than a stale or zero-value coordinate; the UI must not silently fabricate
  a location, and templates requiring GPS evidence can be configured to accept network-based
  coarse location as a fallback with a lower `accuracy_meters` value clearly surfaced, or to block
  submission entirely pending location resolution, per tenant policy.
- **FR-EVD-105 — Evidence captured offline then object deleted before sync.** If a user deletes a
  locally-captured evidence item (device storage pressure, user action) before the Mobile/Offline
  sync queue has uploaded it, the sync queue entry for that evidence is discarded client-side and
  the server never learns it existed — this is intentional per Mobile/Offline module ownership of
  the local queue. If the evidence was already referenced by a field marked mandatory on its
  parent Submission, the Submission fails structural validation on sync (missing mandatory
  evidence) and surfaces as a standard field-validation error, not an evidence-specific error.
- **FR-EVD-106 — Late-arriving evidence after parent already `VERIFIED`.** Evidence that syncs
  after its parent WorkItem/Submission has already reached a terminal verified state (delayed
  offline sync) is still accepted and linked, but the parent record is not reopened or
  re-scored automatically; it is flagged for manager awareness (`late_evidence = true` on the
  link) since source-spec §17.4/P7 treats completed records as historically immutable against the
  template version used.

## 14. Non-Functional Requirements

| Requirement | Target | Notes |
|---|---|---|
| Upload throughput | Sustain burst uploads from a single large site (e.g., 200 concurrent associates at shift start) without server-side queuing delay, since bytes go direct-to-object-storage | Signed-URL issuance itself must meet platform API p95 < 300 ms (source-spec §23) |
| Media processing latency | Photo thumbnail available within 5s p95 of upload-complete; video proxy/transcode within 60s p95 for a 120s clip | Longer-running video jobs surfaced via `large-file-compression-in-progress` state, not a blocking UI wait |
| Retention-enforcement SLA | Retention sweep evaluates and actions every eligible record within 24 hours of crossing its retention boundary, subject to legal hold | Sweep is a scheduled background job, not real-time |
| Hash verification latency | Upload-complete call (including hash verification) completes within 2s p95 for objects up to 25MB | Larger objects verified asynchronously with `UPLOADED → PROCESSING` transition not blocking on full re-hash |
| Durability | Object storage durability per underlying S3-compatible provider SLA (typically 11 nines); metadata durability per Postgres HA/backup configuration | No evidence-specific replication beyond platform standard |
| Availability | Evidence read path (signed URL issuance + CDN-fronted thumbnails) meets platform 99.9%+ core API target | Evidence Service degrades read-only under object-storage partial outage (see high-level-design.md §8) |

## 15. Dependencies

- **Mobile/Offline (EPIC-05):** Owns local capture UX, on-device queueing, retry scheduling, and
  background upload execution. Evidence Management owns the server contract those uploads target
  and the local↔server identity mapping (`client_evidence_uuid`).
- **Forms (EPIC-04):** Owns which template fields require which evidence types and field-level
  mandatory/conditional rules. Evidence Management enforces evidence-object integrity/lifecycle
  once a field references an evidence item, and reports missing-mandatory-evidence back as a
  standard field validation error at Submission time.
- **Visual AI (EPIC-10):** Consumes `evidence.uploaded`/`evidence.processed` events for Photo/Video
  evidence and writes back `AI verification result` evidence records (FR-EVD-010) linked to the
  source evidence.
- **Issues & CAPA (EPIC-07):** References Evidence by ID for exception, root-cause, and corrective
  action documentation; Evidence Management's retention-vs-open-Issue interaction (FR-EVD-063)
  requires the Issue module to expose Issue status to the retention sweep.
- **Security & Compliance (EPIC-14):** Provides per-tenant KMS keys, platform-wide encryption
  standards, and the shared `AuditLog`/tamper-evident audit trail that Evidence Management writes
  into.
- **Analytics (EPIC-15):** Consumes evidence metadata (not raw bytes) via CDC for
  evidence-backed-completion metrics (source-spec Capability Map: "Evidence-backed completion").

## 16. Resolved Decisions

- **Signature vector-stroke retention (FR-EVD-005)**: Vector-stroke data is retained under the
  **same retention class as its rendered PNG**, never independently or indefinitely. The two are
  captured together, always linked, and legally constitute one evidentiary unit, so splitting their
  retention would create a state where one representation of a signed record outlives the other —
  an audit-defensibility gap this platform does not accept.
- **Storage tiering thresholds (hot → cold transition)**: Default **90 days** since last access, as
  already specified in `high-level-design.md` §7, tenant-configurable within a **30–180 day** band
  for tenants with atypical audit-recall patterns. 90 days is the firm default (not merely a
  proposal) because it comfortably exceeds the typical audit-recall window (most corrective/
  verification review happens within 30 days of capture, per source-spec's operational cadence)
  while still moving the bulk of evidence off hot storage well before it becomes a cost driver. The
  default (and the configurable band's bounds) is revisited at the platform's normal annual planning
  cycle using actual access telemetry, the same cadence used for the mobile OS support matrix
  (`engineering-decisions.md` §2).
- **Video audio-track redaction roadmap**: Confirmed out of scope for this version (see FR-EVD-002);
  flagged video audio routes to the manual human review queue described there. No further
  audio-analysis capability is assumed or planned within this spec package; if a future Visual AI
  capability adds audio analysis, it is a separate module-level scope addition, not an extension of
  this decision.
- **Barcode/QR evidence reference-data validation (FR-EVD-009)**: Evidence Management persists only
  the decoded scan payload as proof of capture; it does not validate the payload against any
  product/asset/location catalog. Resolving a scanned code to a catalog record is the Forms/
  Reference-Data path's responsibility (consistent with Frontline Mobile & Offline's own handling of
  unresolved scans, which queues them for resolution on next sync rather than validating on-device).
  This division is firm: Evidence Management owns custody and integrity of what was captured, never
  business-data validation of its contents.
- **Orphan-evidence grace period and quota warning thresholds (FR-EVD-101, FR-EVD-103)**: The 7-day
  orphan-evidence grace period and the 80%/95% quota warning thresholds are the firm platform
  defaults, tenant-configurable. 7 days is chosen because it comfortably covers a frontline worker's
  return-from-days-off window to correct and resubmit a rejected submission without needing to
  recapture evidence; 80%/95% follows the platform's standard two-stage (warn/block) capacity-alert
  pattern used elsewhere (e.g., mobile pending-evidence budget warnings, Frontline Mobile & Offline
  functional-spec §4) so admins see one consistent alerting shape platform-wide.
