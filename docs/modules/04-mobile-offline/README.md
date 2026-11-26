# Module 04 — Frontline Mobile & Offline

> The mobile app itself — for most frontline workers, the only surface of the platform they ever touch. It shows each person their work (Today/Next), captures photos, scans, and signatures camera-first, and keeps working fully offline for weeks, syncing safely later through a retry-safe protocol. It exists because frontline work happens in stockrooms, basements, and dead zones where connectivity cannot be assumed.

| | |
|---|---|
| **Module code** | None assigned in the spec (identified by folder `04-mobile-offline`) |
| **Owning service** | React Native mobile client + server-side Sync Service (the mobile-facing façade over Work/Form/Evidence services) |
| **Epics** | EPIC-05 |

## In plain terms
- Open the app and see your urgent/due/overdue work instantly — rendered from the phone's local database, no network call, even mid-audit in airplane mode.
- Capture evidence camera-first: tapping a photo field opens the viewfinder directly, and every capture is written to encrypted local storage atomically, so an app kill or dead battery never loses a shot.
- Scan barcodes and QR codes as a fast path — for form fields or just to "scan to find work for this asset" — with a manual-entry fallback for damaged labels.
- Stay offline for at least 14 days of normal use: everything queues locally, then syncs in the background when connectivity returns, with resumable uploads for big videos.
- Resolve conflicts gracefully: if two people edited the same task offline, the app shows both values per field with a "Keep mine / Use server's" choice — submissions themselves can never conflict.
- Stay secure and current: biometric app unlock, encrypted local data, screen-capture prevention on sensitive evidence, and a blocking "update required" gate when a client version is too old.

## Key concepts
| Concept | What it means |
|---|---|
| WatermelonDB | The on-device local database (SQLite-based) — a bounded, TTL'd cache of work, forms, permissions, and locations, never a full replica. |
| sync_queue | The outbound queue of unsynced local changes, drained by the background sync engine with retry and backoff. |
| sync_cursor | The per-device bookmark the server uses to hand back only what changed since last sync (delta sync, typically under 50 KB). |
| client_uuid | The idempotency key on every submission — a retried sync of the same work returns the original result instead of creating a duplicate. |
| Bootstrap sync | The one-time, paginated initial download that requires connectivity; after it, every session works fully offline. |
| Conflict resolution | Per-field side-by-side choice for mutable records edited on two devices; server is always the source of truth, client clocks are never trusted for ordering. |
| Pinned template versions | Cached form schemas still needed by unfinished local work are never evicted, so an interrupted fill always resumes against the same form. |
| Device posture | Signals like jailbreak/root detection reported at sync; tenant policy can block sync (never local capture) from non-compliant devices. |

## How it connects to the rest
- Caches what Identity & Organization (01), Forms (03), and Communications/Knowledge (07) serve it — including a pre-resolved permission snapshot it enforces but never recomputes.
- Reflects local work transitions (`AVAILABLE → IN_PROGRESS → SUBMITTED`) up to Work Orchestration (02), which owns scheduling, SLA, and approvals once work is submitted.
- Captures evidence and uploads it via resumable multipart directly to Evidence Management's (05) signed-URL endpoints; Evidence owns the bytes from the moment they land server-side.
- Receives push notifications from the Notification Hub (as a hint to sync, never a correctness dependency) and schedules its own local notifications offline; all domain events are emitted server-side by the Sync Service, never by the phone.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-MOB-001–054: the Today/Next/Manager surfaces, cold-start budgets, camera-first capture, scanning, one-handed UX, accessibility, push/local notifications, upgrade resilience, device security, and the offline behavior contract. |
| [high-level-design.md](./high-level-design.md) | The device-side architecture (UI, local DB, sync engine, native bridges) plus the Sync Service, the per-table cache freshness/eviction policy, the client-pull sync scaling model, and single-tenant-per-session on-device rules. |
| [low-level-design.md](./low-level-design.md) | The complete WatermelonDB schema (users, permissions, work_items, submissions, evidence, sync_queue, cursors), the `/api/v1/sync` contract, bootstrap/push-pull/media-upload sequences, conflict merging, and eviction algorithms. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: device + Sync Service component diagram, new-device bootstrap, offline capture-to-sync with a conflict, resumable media upload with interruption, local ER diagram, and the sync/connectivity state machine where offline is a normal state. |
