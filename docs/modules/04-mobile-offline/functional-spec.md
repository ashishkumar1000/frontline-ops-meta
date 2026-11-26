# Frontline Mobile & Offline — Functional Specification

Module corresponds to **EPIC-05 (Mobile Offline)**. Source: source-spec.md §9 (Frontline Mobile
and Offline), §7.3 (Work Item UX "Today" screen), §19 (Offline Sync and Conflict Resolution, all
subsections), §18.2 (Sync Endpoint), §4 Principles P1/P2/P9, Appendix E EPIC-05. This is the
product's core strategic bet — "mobile-first" and "offline is normal" (P1, P2) — and is treated
here at maximum depth.

## 1. Purpose and Scope

### 1.1 Purpose
The Frontline Mobile & Offline module is the primary — for most frontline users, the *only* —
surface of the platform. It must let a Frontline Associate, Auditor/Inspector, or Store/Site
Manager know what to do next and complete it in seconds, with a camera-first, thumb-driven
interaction model, regardless of network connectivity, and later reconcile that work with the
server through a protocol that is safe to retry, safe to interrupt, and safe to run from multiple
devices. This module is not "the app that happens to also work offline" — offline execution is a
first-class, permanently supported operating mode (source-spec §9.2 offline design rule), tested,
monitored, and designed for with the same rigor as the online path.

### 1.2 In Scope
- The React Native mobile client: app shell, navigation, and the Today/Next/Manager surfaces
  (§7.3) rendered from local cache.
- Fast start/resume behavior and cold/warm start performance targets.
- Camera-first evidence capture UX for every evidence type owned by the Evidence module (photo,
  video, audio, document scan, signature) as they are captured *on this client*.
- Barcode/QR scanning UX (as a form field type and as a fast lookup action).
- One-handed interaction design: thumb-zone layout, bottom-sheet primary actions, reachability.
- Accessibility: screen reader support, dynamic font scaling, contrast, motor-impairment support.
- Push notification receipt/display and local (on-device, offline-queued) notification scheduling.
- App upgrade resilience: local DB schema migration across app versions, forced-upgrade gating.
- Device-level security controls: biometric app-unlock, MDM/device-posture signal consumption,
  screen-capture prevention for sensitive evidence.
- The on-device local database (WatermelonDB) as a bounded, TTL'd cache of forms, permissions,
  locations, assigned work, and knowledge (§19.1).
- The idempotent offline sync protocol between client and server (§18.2, §19.2–19.4), including
  the server-side Sync API that serves this client.
- Conflict detection, conflict UX, and conflict resolution semantics for locally-modified records.
- Background sync scheduling, battery/data budget, and connectivity-aware sync heuristics.

### 1.3 Out of Scope (owned by other modules, referenced here)
- Organization hierarchy, role/scope computation — **Identity/Org module (EPIC-01/02)**. This
  module caches the *resolved effective permission set* it is handed; it does not compute RBAC/ABAC.
- Form schema design, validation logic, template authoring — **Forms module (EPIC-04)**. This
  module caches published `TemplateVersion` schemas and renders/validates them locally using rules
  shipped inside the schema payload.
- WorkItem state machine, scheduling, assignment logic — **Work Orchestration module (EPIC-03)**.
  This module caches assigned `WorkItem` rows and reflects local state transitions
  (`AVAILABLE → IN_PROGRESS → SUBMITTED`) but does not own escalation, SLA timers, or approval
  routing.
- Evidence media processing, hashing, virus scan, retention, AI verification — **Evidence module
  (EPIC-06)**. This module owns *capture* and *upload transport*; the Evidence Service owns
  storage, integrity, and lifecycle once a media object lands server-side.
- Issue/CAPA lifecycle — **Issues & CAPA module (EPIC-07)**; this client only renders issue
  WorkItems and captures corrective-action evidence.
- Notification delivery infrastructure (push fan-out, preference resolution, retries) —
  **Notification Hub**; this module is a push *consumer* (device token registration, receipt
  handling, in-app display) and a local-notification *scheduler*.
- Knowledge document content, versioning, and search ranking — **Communications/Knowledge module
  (EPIC-08)**; this module caches a bounded, audience-scoped slice for offline reading.
- Web Admin Console — **out of scope entirely**; this module is mobile-only (iOS/Android).

## 2. Personas Served

| Persona | Primary use of this module |
|---|---|
| Frontline Associate (primary) | Opens app dozens of times/shift; Today/Next screens; camera-first evidence capture; barcode scans; offline execution during connectivity gaps (stockrooms, basements, rural sites, moving vehicles) |
| Auditor/Inspector | Multi-hour, multi-location audit sessions, frequently in airplane mode or cellular dead zones (warehouses, plants, remote sites); heavy media capture volume per session; needs guaranteed zero-loss capture |
| Store/Site Manager (on mobile) | Manager Workspace tab (Exceptions/Approvals/Team/Location health) on the same app; approves/reassigns from mobile between other duties; lower media-capture volume, higher read/approve volume |

## 3. Functional Requirements

### 3.1 App Shell & Navigation

- **FR-MOB-001** — The app root navigation exposes three top-level destinations matching
  source-spec §7.3: **Today**, **Next**, and **Manager** (Manager tab visible only to principals
  whose cached `permissions` include a manager-scope role for at least one assigned location).
  Navigation state (last active tab, scroll position, in-progress draft) survives app kill/restart.
- **FR-MOB-002** — **Today** renders, entirely from local cache, three live counters — *urgent*,
  *due*, *overdue* — computed client-side from cached `work_items` against device wall-clock, and
  a scrollable list of today's WorkItems grouped by urgency band. Counters must render within the
  cold-start budget (§7) with zero network calls.
- **FR-MOB-003** — **Next** renders an ordered queue of the user's next 1–N actionable WorkItems
  (default N=3 visible, expandable), ranked by: overdue > due-soonest > highest configured
  priority > creation order. Tapping an item opens it directly into the capture/submission flow
  (no intermediate detail screen for single-field or short-form WorkItems).
- **FR-MOB-004** — **Manager** exposes four sub-views — Exceptions, Approvals, Team, Location
  health — as summarized cards with drill-down; Manager views may show a "stale" badge with
  last-cache-refresh timestamp when data is older than the freshness policy in
  `high-level-design.md` §4.
- **FR-MOB-005** — All three top-level screens must render a fully populated, non-blank state from
  local cache alone, with no loading spinner gating first paint, even on first app open after
  install once initial bootstrap sync (LLD §5, Sequence A) has completed once.

### 3.2 Fast Start / Resume

- **FR-MOB-006** — Cold start (process not resident) to interactive Today screen: **p95 ≤ 2.0s** on
  the reference-low device (see NFR table, §6). Warm resume (app backgrounded, process
  resident): **p95 ≤ 300ms**.
  - **Reference device tiers (resolved)**: Two fixed benchmark tiers are defined. **Reference-low**
    is the release-gating tier, defined as the platform's own minimum-supported hardware floor from
    `engineering-decisions.md` §2 (Android: quad-core CPU, 2GB RAM, API 29; iOS: iPhone 8 / A11
    Bionic, iOS 16) — this tier is a fixed device class re-selected only when the platform's minimum
    hardware floor itself changes at an annual planning cycle, not benchmarked against fleet
    telemetry, so the low-end target cannot silently drift upward over time. **Reference-mid** is
    the median device model observed in aggregate tenant device-fleet telemetry over the prior
    quarter, re-benchmarked quarterly. The §6 NFR targets (cold start p95 ≤ 2.0s, warm resume p95 ≤
    300ms) must be met on **reference-low**, which is the release gate; reference-mid results are
    tracked for trend visibility only. Gating on the low tier, not the fleet median, is the firm
    decision because this platform's stated differentiator is low-end-device performance
    (`engineering-decisions.md` §2), so the floor device, not the average device, must define the
    release bar.
- **FR-MOB-007** — On resume, the app must re-validate session/token freshness asynchronously,
  without blocking first paint; if the token is expired and cannot be silently refreshed, the app
  still shows cached Today (read-only banner state, see §5) rather than forcing a blocking login
  screen.
- **FR-MOB-008** — An in-progress capture flow (camera open, form draft, scan-in-progress) that is
  interrupted by an OS-level app kill (memory pressure, incoming call, user swipe-kill) must be
  fully recoverable on next launch: draft submission state, captured-but-unattached media, and
  scroll/field-focus position are restored from the local DB write-ahead state (§9 edge cases).

### 3.3 Camera-First Evidence Capture

- **FR-MOB-009** — Every evidence-bearing form field (Photo, Video, Audio, Document upload,
  Signature — source-spec §8.1) opens directly into a capture-first UI: the device camera/mic/sign
  pad is the default first screen for that field, not a menu of "take photo / choose from
  library / cancel". A secondary, less prominent affordance allows choosing from the existing
  gallery/library where the template permits it (`allow_library_import: boolean` on the field
  schema; defaults `true` for Photo/Video/Document, `false` for Signature, tenant-configurable).
- **FR-MOB-010** — Photo capture: default to the rear camera, in-viewfinder grid/level overlay
  (configurable per template — e.g. shelf-compliance templates default the level overlay on),
  multi-shot burst mode for fields marked `multi: true` (e.g. "photograph all four corners"),
  inline retake before confirm, and mandatory-field visual indicator (red border) persists until a
  valid capture is attached.
- **FR-MOB-011** — Video capture: max duration and max file size are per-field schema attributes
  (`max_duration_sec`, `max_size_mb`); the capture UI shows a live countdown once past 80% of the
  duration budget and auto-stops at the limit rather than producing an oversized/rejected file.
- **FR-MOB-012** — Audio capture: waveform-visible recorder, pause/resume, and playback-before-
  confirm; used for voice-note evidence and voice-to-text-assisted free text fields (Phase 2 flag).
- **FR-MOB-013** — Signature capture: full-screen pad, captures device-relative timestamp and (if
  location permission granted) GPS at time of signature; signature evidence is immutable once
  confirmed — re-signing creates a new evidence record, it does not overwrite.
- **FR-MOB-014** — Every captured evidence item is written to the local `evidence` table
  (state `PENDING_UPLOAD`) and to on-device encrypted file storage **atomically with the capture
  action** — before the user proceeds to the next field — so that app kill immediately after
  capture never loses the media (see §9, "app killed mid-capture").
- **FR-MOB-015** — GPS and device metadata (device model, OS version, app version, local
  timestamp, timezone offset) are attached automatically to every evidence record and to every
  submission at capture time and at submit time respectively; GPS capture failure (permission
  denied, no fix within timeout) degrades to "location unavailable" rather than blocking capture,
  unless the field is explicitly configured `gps_required: true`.
- **FR-MOB-016** — Document upload fields support in-app document scanning (edge-detection,
  perspective-correct, multi-page) as the camera-first path, with "import from files" as secondary.

### 3.4 Barcode / QR Scanning

- **FR-MOB-017** — A dedicated, persistent **Scan** quick action is reachable from the bottom
  thumb-zone on Today/Next (not buried in a menu) for fast product/asset/location lookup
  independent of any specific form (e.g., "scan to find work for this asset").
- **FR-MOB-018** — Barcode/QR form fields (source-spec §8.1) open the scanner directly; supported
  symbologies: QR, Code128, Code39, EAN-13/8, UPC-A/E, PDF417, Data Matrix. Scanner auto-detects
  symbology; no manual symbology selection required.
- **FR-MOB-019** — Successful scan auto-advances to the next field/screen (no separate confirm
  tap) unless the field is configured `require_confirm: true` (used where a scanned value drives a
  branching/mandatory rule the user should visually verify).
- **FR-MOB-020** — Scan failure/timeout (poor lighting, damaged code) after 8 seconds surfaces a
  manual entry fallback (numeric keypad for numeric symbologies) without leaving the field.
- **FR-MOB-021** — A scanned value resolved against cached reference data (product/location/asset
  lookup field type) that is **not found in local cache** is queued as an unresolved lookup and
  resolved on next sync; the field shows the raw scanned code with a "pending verification" badge
  rather than blocking submission.

### 3.5 One-Handed Interaction

- **FR-MOB-022** — Primary actions (submit, next field, confirm capture, scan) are placed in the
  bottom third of the screen (thumb zone) using bottom sheets and bottom action bars; destructive
  or rarely-used actions (delete draft, discard, admin overrides) are placed top-of-screen or
  behind a secondary affordance, never in the thumb zone.
- **FR-MOB-023** — Forms with more than one field per screen use a single-column, one-field-
  emphasized layout with swipe-forward/back gesture navigation between fields, in addition to a
  tappable progress indicator, so a form can be completed with one thumb without reaching across
  the screen.
- **FR-MOB-024** — All interactive targets meet a minimum 44x44pt (iOS) / 48x48dp (Android) hit
  area regardless of visual size, per platform HIG/Material guidance.
- **FR-MOB-025** — Large-device (tablet) layouts reflow to a two-pane layout (list + detail) but
  preserve bottom-anchored primary actions on the detail pane for one-handed tablet holding
  patterns common among auditors.

### 3.6 Accessibility

- **FR-MOB-026** — Full screen-reader support (VoiceOver/TalkBack): every interactive element has
  an accessible label, role, and state; form field validation errors are announced on focus, not
  only shown visually.
- **FR-MOB-027** — Dynamic type/font scaling up to the platform's largest standard accessibility
  size without truncation or overlap on Today/Next/capture screens; layouts use flexible, not
  fixed-height, text containers.
- **FR-MOB-028** — Color is never the sole signal for state (urgent/overdue/error/success); each
  state pairs color with an icon and/or text label. Minimum contrast ratio 4.5:1 for body text,
  3:1 for large text/icons (WCAG 2.1 AA, per source-spec §23 "platform accessibility patterns on
  mobile").
- **FR-MOB-029** — Motor-impairment support: all gesture-based interactions (swipe between fields,
  swipe-to-dismiss) have an equivalent single-tap alternative; no interaction requires a timed
  gesture or multi-finger gesture as the *only* path.
- **FR-MOB-030** — Camera/capture screens support an assistive "voice-triggered capture" (Phase 2,
  feature-flagged) for users who cannot reliably tap the shutter.

### 3.7 Push & Local Notifications

- **FR-MOB-031** — The app registers a device push token (FCM for Android, APNs for iOS) with the
  Notification Hub on login and on token rotation, associated to `device_id` + `user_id` +
  `tenant_id`; token registration is retried with backoff if it fails and is not itself gated on
  connectivity of any other feature.
- **FR-MOB-032** — Push notifications for WorkItem assignment, due-soon, overdue-escalation,
  mandatory-acknowledgement announcements, and issue-assigned events deep-link directly into the
  relevant WorkItem/detail screen from cold start, cached-first (no network wait) if the target
  entity is already in local cache; otherwise a bounded fetch-then-render with a loading state.
- **FR-MOB-033** — **Local notifications** (device-scheduled, no server/network dependency) are
  used for: (a) due-soon reminders computed from cached `work_items.due_at` even while fully
  offline; (b) "you have N items pending upload — reconnect to sync" reminders when the outbound
  `sync_queue` has been non-empty for longer than a configurable threshold (default 4 hours); (c)
  low-battery-during-active-audit warnings (§3.9). Local notifications are recomputed/rescheduled
  every successful cache refresh and on app foreground.
- **FR-MOB-034** — Notification preferences (quiet hours, category opt-out) are cached locally and
  respected by the local-notification scheduler even offline; push suppression during quiet hours
  is enforced server-side by the Notification Hub and is out of scope here.
- **FR-MOB-035** — Duplicate push delivery (retry from Notification Hub) must not create duplicate
  local state changes or duplicate local notifications; the client de-dupes by the event's
  `event_id`.

### 3.8 App Upgrade Resilience

- **FR-MOB-036** — The local WatermelonDB schema is versioned independently of the app's
  marketing/build version. Every app release that changes the local schema ships a migration
  script (additive-first: add columns/tables, backfill, then remove in a later release) run
  automatically and transactionally on first launch after upgrade, before any UI renders.
- **FR-MOB-037** — Migration failure (corrupt DB, unexpected pre-migration state) triggers a
  **local data reset with recovery**: any un-synced outbound data in `sync_queue`/`submissions`/
  `evidence` (state != synced) is exported to a local quarantine file and the user is shown a
  blocking recovery screen offering (a) retry migration, (b) contact support with a diagnostic
  bundle, before allowing continued use; a full destructive reset is never silent.
- **FR-MOB-038** — **Forced-upgrade gating**: the server can mark a minimum supported app version
  per platform (tenant-overridable). On app foreground/launch, the client checks this against its
  own version via a lightweight version-check call (or a cached value with a max staleness of 24h
  when offline); below-minimum versions are shown a **blocking "update required" screen** (§5)
  that still allows viewing already-cached read-only Today data but blocks new capture/submission
  and sync, to prevent a stale client from writing schema/contract-incompatible payloads to the
  Sync API.
- **FR-MOB-039** — A "soft" recommended-upgrade banner (dismissible, non-blocking) is shown between
  the recommended and minimum version, distinct from the blocking forced-upgrade screen.
- **FR-MOB-040** — App store review/release rollback: because published `TemplateVersion` schemas
  are immutable and versioned server-side, an app that is one minor version behind must still be
  able to render and submit against the current published template versions its schema-rendering
  engine understands; the rendering engine's own capability version is advertised in the sync
  handshake so the server can withhold field types the client cannot yet render (graceful
  degradation, not a hard block) where policy allows.

### 3.9 Device Security

- **FR-MOB-041** — Optional, tenant-configurable **biometric app-unlock** (Face ID / fingerprint)
  as a second factor gating app foreground-resume after a configurable inactivity timeout (default
  5 minutes), independent of the OS-level device lock. Falls back to device PIN/passcode entry
  when biometrics are unavailable/disabled/failed 3x.
- **FR-MOB-042** — The app consumes MDM/device-posture signals (jailbreak/root detection, OS
  version floor, managed-device attestation via platform APIs — Android SafetyNet/Play Integrity,
  iOS DeviceCheck-class signal) and reports posture on each sync handshake; a tenant policy can be
  configured to block sync (not app use) from devices failing posture checks, surfaced as a
  distinct UX state (§5).
- **FR-MOB-043** — **Screen-capture prevention** is applied to screens displaying sensitive
  evidence (e.g., configured-confidential audit findings, signature capture, PII-bearing lookup
  results) using platform APIs (`FLAG_SECURE` on Android; screen-recording detection + content
  obscuring on iOS) on a per-tenant/per-template `sensitive: true` flag.
- **FR-MOB-044** — All locally cached data (WatermelonDB file, evidence media staging area,
  encrypted token/credential store) is encrypted at rest using platform-native secure storage
  (Android Keystore-backed SQLCipher, iOS Keychain + Data Protection class `NSFileProtectionComplete`
  for the DB file and media directory).
- **FR-MOB-045** — Logout wipes all locally cached tenant/user data **except** unsynced outbound
  work, which requires explicit user confirmation ("You have N unsynced items — sync now, or
  discard and log out?") before destructive logout is allowed (see §9, "logout while items
  queued").
- **FR-MOB-046** — Session/refresh tokens are never persisted in WatermelonDB tables (which may be
  exported for diagnostics); they live only in platform secure storage.

### 3.10 Offline Behavior

- **FR-MOB-047** — The following functions work with **zero connectivity**, indefinitely, bounded
  only by local storage and cache freshness: viewing Today/Next/Manager cached views; opening and
  completing any cached `TemplateVersion` form; camera/video/audio/signature/document capture;
  barcode/QR scanning against cached reference data; submitting a WorkItem (queued locally);
  viewing cached Knowledge documents; viewing cached assigned-work detail and history; local
  notification scheduling/delivery.
- **FR-MOB-048** — The following functions **require connectivity** and degrade explicitly rather
  than silently failing: initial bootstrap (first-ever cache population, FR-MOB-049); live
  Manager-view data beyond cache freshness window; Knowledge search beyond the cached document set
  (falls back to local full-text over cached documents only); real-time chat/collaboration
  features (not in this module); AI/Visual-AI verification results (evidence uploads and shows
  `PENDING_VERIFICATION` until connectivity returns).
- **FR-MOB-049** — A brand-new device/install cannot use the app fully offline before at least one
  successful bootstrap sync while online; the first-run flow explicitly requires connectivity and
  communicates this (distinct from all subsequent sessions, which work fully offline).
- **FR-MOB-050** — Local storage budget: default cap 500MB per tenant-user profile for reference
  data + metadata (configurable per tenant), plus a separate, larger pending-evidence media budget
  (default 2GB, tenant-configurable, warns at 80%) that is *not* subject to LRU eviction (evidence
  belongs to the user's own unsynced work and is never silently deleted — see LLD §7 eviction
  policy).
- **FR-MOB-051** — Background sync attempts opportunistically on any network transition to
  connected, and on a periodic schedule (default every 15 minutes on Wi-Fi/unmetered, every 60
  minutes on cellular, configurable) via WorkManager (Android) / BGTaskScheduler (iOS), subject to
  OS-level background execution budgets; the app never assumes precise background timing on iOS
  and always offers a manual "Sync now" affordance.
- **FR-MOB-052** — Data-usage budget: media upload over cellular defaults to **Wi-Fi-only** for
  video (tenant/user-configurable per connection type), while form/metadata sync (small JSON
  payloads) proceeds over cellular by default; users can override per-session ("upload now over
  cellular").
- **FR-MOB-053** — Battery budget: background sync backs off (longer interval, smaller batch) below
  a device battery threshold (default 15%) unless the device is charging; foreground/manual sync is
  never blocked by battery level, only warned (§5 low-battery capture warning applies to *active
  capture sessions*, not background sync).
- **FR-MOB-054** — Supported continuous offline session duration: **at least 14 days** of normal
  frontline usage (WorkItem completion, evidence capture) without requiring reconnection, bounded
  by local storage budgets, not by an artificial time limit; the app never forces a logout or wipes
  local state purely due to offline duration (session/token expiry is handled per FR-MOB-007, not
  by data loss).

## 4. UX States (Exhaustive)

| State | Trigger | Behavior |
|---|---|---|
| **Offline banner — informational** | Device reports no network | Persistent, non-blocking top banner: "You're offline. Work is saved and will sync automatically." All capture/submit flows remain fully enabled. |
| **Offline banner — queued count** | `sync_queue` non-empty while offline | Banner extends to show count: "N items waiting to sync." |
| **Sync-in-progress** | Background or manual sync running | Small animated indicator in the banner/header; does not block navigation or capture; a manual "Sync now" tap while already syncing is a no-op (button shows spinner, disabled). |
| **Sync succeeded (transient)** | Sync completes with all items accepted | Banner briefly (3s) shows "All synced" then reverts to the default online (no-banner) state. |
| **Sync-failed-needs-attention** | One or more items in `sync_queue` moved to `FAILED` after exhausting retry policy (LLD §8) for a non-conflict, non-transient reason (e.g. `413`, persistent `4xx` validation) | Non-dismissible (until acted on) banner: "N items couldn't sync — tap to review." Drill-in screen lists each failed item with the machine-readable reason in plain language and a Retry / Edit / Discard action per item. |
| **Storage-nearly-full** | Reference-data cache or pending-evidence budget crosses 80% | Dismissible warning banner + entry in Settings; at 95% pending-evidence budget, **new large-media capture is blocked** with an explicit modal ("Free up space or connect to sync pending uploads") rather than allowed to silently fail mid-capture; reference-data cache at 95% triggers eviction (LLD §7) rather than blocking, since it is reconstructable. |
| **Conflict-needs-resolution** | Server returns a `conflict` entry in a sync response for a locally-modified mutable record (task/issue field-level conflict, not immutable submissions) | Item is removed from the normal Today/Next flow and placed in a dedicated "Needs your review" list; opening it shows a side-by-side (local value vs. server value) per conflicting field with a per-field "Keep mine / Use server's" choice and a combined "Save resolution" action that re-submits as a new, explicit change. |
| **Low-battery capture warning** | Battery < configurable threshold (default 10%) while an active multi-step capture session (e.g., a long audit) is open | Non-blocking toast/banner: "Low battery — your progress is saved after every step" (reassurance, not restriction); does not block capture, since forced early submission would be worse than a low-battery warning. |
| **Permission-denied — camera** | OS camera permission denied/revoked | Capture screen shows an explanatory state ("Camera access is needed to capture photo evidence") with a direct deep link to OS Settings; the field remains mandatory/blocked until resolved or, if the template allows, a manual "note reason" fallback text field appears (tenant/template-configurable `allow_evidence_waiver_on_permission_denied`). |
| **Permission-denied — location** | OS location permission denied/revoked | GPS metadata is omitted, field shows "Location unavailable — permission denied"; does not block submission unless the field is `gps_required: true`, in which case the same explanatory + Settings-deeplink pattern as camera applies. |
| **App-update-required (blocking)** | Client version below tenant-configured minimum (FR-MOB-038) | Full-screen blocking state over new capture/submit/sync actions: "An update is required to continue." with a store-deeplink "Update now" button; cached Today list remains visible in read-only mode beneath/behind an explanatory card so the user isn't left with a blank screen. |
| **Bootstrap-in-progress (first run)** | New device install, first login | Full-screen progress state with staged messaging ("Setting up your work…", "Downloading your forms…", "Almost ready…") mapped to bootstrap phases (LLD §5 Sequence A); requires connectivity; a connectivity loss mid-bootstrap pauses and resumes rather than restarting from zero. |
| **Empty — no assigned work** | Today/Next cache legitimately has zero items | Friendly empty state, distinct from a loading/error state, with a manual refresh affordance. |
| **Stale-manager-data** | Manager view data older than freshness policy while offline/degraded | "Showing data as of [timestamp] — reconnect for latest" inline note on the relevant card, not a full-screen block. |
| **Device-posture-blocked** | MDM/posture check fails per tenant policy (FR-MOB-042) | Sync is blocked with an explanatory screen; local capture/queueing still works (data is not lost) so the user is not penalized for a device-compliance issue outside their control until IT resolves it. |

## 5. Validation & Edge Cases

| Edge case | Required behavior |
|---|---|
| Device clock skew (client clock materially wrong) | Client still records `created_at_client`/`modified_at_client` for local ordering and UX display, but the server assigns and treats its own received-at timestamp as authoritative for all cross-device ordering, SLA computation, and conflict "who changed it last" logic (source-spec §19.4). Client clock is never trusted for authoritative ordering. |
| App killed mid-capture (photo/video/signature in progress) | Media captured before the kill is already persisted (FR-MOB-014); on relaunch the draft submission resumes at the same field with the already-captured evidence intact; only in-flight (not-yet-confirmed) capture is lost, and the user re-attempts that one capture. |
| Storage full during media capture | Capture UI pre-checks available space before opening the camera for video (which can be large) and blocks with an explicit "not enough storage" modal rather than starting a recording doomed to fail; for photo (small), a failed write is caught and surfaced immediately with a retry, not silently dropped. |
| Going online mid-sync (connectivity flips during an in-flight sync batch) | The sync engine's push/pull loop (LLD §6) is chunked and checkpointed; a mid-batch disconnect leaves already-acknowledged items marked synced and unacknowledged items back in `PENDING`/`QUEUED` for the next attempt — never double-applies and never loses the local copy of an unacknowledged item. |
| Logout while items queued | Blocking confirmation dialog (FR-MOB-045) naming the queued count; "Sync now, then log out" attempts one final sync pass with a timeout before proceeding; "Discard and log out" requires a second, explicit confirmation and is logged as a local audit event for support diagnosis. |
| Multi-device, same user | Each device has its own `device_id` and independent `sync_cursor`; the server is the single source of truth, so a submission created on device A appears as an incoming `server_change` on device B on its next sync. Drafts (unsubmitted, in-progress work) are **not** synced/shared across devices in v1 — draft state is local-only until submitted. Cross-device draft handoff is deferred past MVP (see §8) given the added conflict-surface complexity of syncing partial, field-level draft state across devices. |
| Duplicate submission (retry of an already-accepted submission) | Server returns 200 with the original server identity per source-spec §19.4; client treats this as success, not as an error, and clears the item from `sync_queue`. |
| Two users edit the same mutable task offline, both come online | Field-level values that differ produce a `conflict` entry per source-spec §19.3; fields that don't differ merge silently. See LLD §6 for the merge algorithm. |
| Evidence upload interrupted partway (large video) | Resumable multipart upload (LLD §5 Sequence C) resumes from the last acknowledged chunk on next connectivity, never restarts from zero and never re-uploads a duplicate completed file. |
| User revokes camera permission mid-session (OS settings) | Any already-open capture screen detects the revocation on next foreground and transitions to the permission-denied state (§4) rather than crashing or showing a frozen viewfinder. |
| Tenant force-logs-out a device (admin-initiated deactivation) | Next sync handshake returns an auth-invalidated response; the client preserves unsynced local data in quarantine (same mechanism as FR-MOB-037) and requires re-authentication before resuming — it does not silently discard queued work belonging to a deactivated user, since that work may still need manager recovery/handoff. |
| Corrupted local DB detected outside of a migration (disk corruption, OS crash) | Same quarantine-and-recover flow as FR-MOB-037, triggered by a startup integrity check, not only by migration failure. |

## 6. Non-Functional Requirements

| Requirement | Target | Measurement |
|---|---|---|
| Cold start to interactive Today | p95 ≤ 2.0s on reference-low device (release-gating; see §3.2) | Device telemetry (source-spec §23 "Mobile startup") |
| Warm resume | p95 ≤ 300ms | Device telemetry |
| Sync success rate | ≥ 99% of sync attempts eventually succeed within 24h of connectivity being available; ≥ 95% of standard reconnect syncs complete in < 30s (source-spec §23) | Sync telemetry, per source-spec §26.2 "Reliability" |
| Offline session duration supported | ≥ 14 days continuous offline use without forced logout/data loss | QA soak test + field telemetry |
| Zero data loss on kill/restart in airplane mode | 100% — this is a release-gating acceptance test (source-spec §24.1 "Offline submission") | QA acceptance suite |
| Local storage footprint | Reference-data cache ≤ 500MB default budget; pending-evidence budget configurable, default 2GB, never silently evicted | Device telemetry |
| Battery impact of background sync | ≤ 2% of daily battery budget under default sync frequency on reference device | Device telemetry / platform battery attribution APIs |
| Accessibility | WCAG 2.1 AA-equivalent contrast/labeling; full screen-reader operability of Today/Next/capture flows | Automated (axe-equivalent RN linting) + manual audit each release |
| Crash-free session rate | ≥ 99.5% | Crash reporting (Sentry/Crashlytics-class) |

## 7. Dependencies

| Dependency | Direction | Nature |
|---|---|---|
| Identity/Org (EPIC-01/02) | Inbound (consumes) | Cached effective `permissions`/`UserScope` snapshot per bootstrap/refresh; this module does not compute authorization, only enforces the cached decision and re-validates on next sync. |
| Forms (EPIC-04) | Inbound (consumes) | Cached `TemplateVersion` schemas (including validation/branching rules) for offline rendering; submissions reference the exact `template_version_id` used. |
| Work Orchestration (EPIC-03) | Bidirectional | Consumes assigned `work_items`; produces local state transitions (`IN_PROGRESS`, `SUBMITTED`) that sync upstream; WorkItem module owns SLA/escalation once submitted. |
| Evidence (EPIC-06) | Outbound (produces) | This module captures media and metadata and hands off via resumable upload; Evidence Service owns storage, hashing, retention, and AI verification once received. |
| Communications/Knowledge (EPIC-08) | Inbound (consumes) | Cached, audience-scoped Knowledge documents for offline reading; search-relevance and authoring are owned upstream. |
| Notification Hub | Bidirectional | Registers device push tokens; receives push; this module owns local-notification scheduling independently. |
| Sync/Event Backbone | Outbound | Every synced client change ultimately triggers canonical domain events (§18.3) server-side, outside this module's direct ownership. |

## 8. Resolved Decisions

- **Cross-device draft handoff**: Drafts (unsubmitted, in-progress work) remain local-only per
  device for this version; not built now. Syncing partial, field-level draft state across devices
  would require a field-level conflict/merge model for *unsubmitted* work in addition to the
  already-built submitted-record conflict model (§5, §6 of the LLD), which is not justified by
  current requirements. Deferred to a future version if pilot-tenant usage data shows material
  multi-device-per-user prevalence during active drafting.
- **Reference device tiers for performance targets**: Resolved in FR-MOB-006 above — two fixed
  tiers, reference-low (the platform's minimum-hardware floor) as the release-gating benchmark, and
  reference-mid (fleet median, quarterly re-benchmarked) as a trend-only measurement. No further
  segmentation by tenant fleet age is added; a tenant whose fleet is entirely below reference-low
  hardware is out of the platform's supported device matrix per `engineering-decisions.md` §2.
- **Pending-evidence budget overflow override (§4 storage-nearly-full at 95%)**: A per-template
  `never_block_capture: true` flag is supported, defaulting to `false`. When set (intended for
  safety-critical templates such as incident reporting), capture at ≥95% pending-evidence budget is
  never blocked; instead the client automatically degrades capture quality (reduced photo
  resolution / lower video bitrate) to conserve space rather than refusing the capture outright.
  This is firm because for a safety-critical template, a blocked capture is a worse outcome than a
  lower-fidelity one, but the degradation must be automatic and template-scoped rather than a
  blanket platform default, since most templates rely on full-fidelity evidence for compliance
  verification.
- **Voice-triggered capture (FR-MOB-030) and cross-device draft handoff — MVP inclusion**: Both
  remain **out of MVP scope**, staying Phase 2/feature-flagged as already stated in FR-MOB-030 and
  the draft-handoff decision above. Neither has a committed FR or NFR target in this version;
  pulling either into MVP is a scope change to be raised through the normal roadmap process, not a
  documentation ambiguity to resolve here.
- **Biometric app-unlock scope (FR-MOB-041)**: Biometric app-unlock is additive to, not a
  replacement for, the OAuth/OIDC session established via the platform's standard auth flow
  (`engineering-decisions.md` §1) — it gates *app foreground access* on-device only and never
  substitutes for backend authentication or token issuance.
