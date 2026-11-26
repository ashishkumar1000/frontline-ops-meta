# Communications & Knowledge — Functional Specification

Module: `07-communications-knowledge`
Epic: EPIC-08 (Comms & Knowledge)
Source: `source-spec.md` §11.1 (Communications), §11.2 (Knowledge), §17.1 (`KnowledgeDocument`),
Appendix E EPIC-08

## 1. Purpose

This module is the platform's channel for pushing governed information down to frontline users
(Communications) and the authoritative, authorization-aware store of operational knowledge that
frontline users, managers, and AI agents retrieve from (Knowledge Hub). Both halves share one
concern: **getting the right information, to the right scoped audience, provably** — provable
delivery/read/ack for announcements, provable authorization-consistent retrieval for knowledge.

The module is the direct upstream dependency for two portfolio agents named in source-spec §13.1:
the **Operations Copilot** (answers operational questions from approved knowledge — consumes this
module's search/RAG API) and the **Communication Agent** (drafts targeted communications — produces
draft `Announcement` records that a human publishes through this module's authoring flow).

## 2. Scope

### 2.1 In scope

**Communications**
- Announcement authoring: title, body (rich text), attachments, priority, category, language(s).
- Audience targeting by organization scope (node subtree), role, and user attribute (e.g.
  `employment_type == 'part_time'`), including intersection/union targeting rules and an audience
  preview tool.
- Mandatory acknowledgement: configurable per announcement, with a blocking UX gate and an
  escalation ladder for non-acknowledgement.
- Multi-channel delivery: in-app feed, push (FCM/APNs via Notification Hub), email, and optional
  SMS/enterprise-messaging (Teams/Slack-class) integration, with channel fallback rules.
- Read receipts (distinct from acknowledgement — "seen" vs. "affirmatively confirmed").
- Scheduled publication and expiry (auto-retraction from active feed at `expires_at`).
- Translation workflow: source-language authoring, machine or human translation into target
  locales, staleness tracking when source content changes after translation.
- Audience preview: "who will receive this" resolution tool shown to the author before publish.

**Knowledge Hub**
- `KnowledgeDocument` lifecycle: draft, review, publish, version, retire (reusing the
  `TemplateVersion`-style versioning pattern from source-spec §17.4, adapted for documents).
- Effective-dated audience scoping per document (and per version) — org scope, role, language.
- Multi-language document versions authored against one logical document.
- Full-text and semantic search over published, authorized documents (OpenSearch + pgvector, per
  conventions.md).
- AI-answer (RAG) endpoint that enforces the same authorization filter as direct document access,
  with mandatory source citation.
- Document ownership, review cadence, and retirement/supersession tracking.
- Offline-cached knowledge access on mobile (bounded reference set per source-spec §9.2).

### 2.2 Out of scope (owned elsewhere, referenced here)

- Identity, roles, `UserScope` resolution mechanics — owned by Identity & Organization (Module 01);
  this module consumes resolved scope to compute announcement audiences and search authorization
  filters, it does not define RBAC/ABAC primitives.
- Push/email/SMS transport, provider failover, delivery retries at the transport layer —
  owned by the platform Notification Hub; this module is a producer of notification requests and a
  consumer of delivery-status callbacks.
- Agent Runtime policy engine, tool allowlisting, autonomy levels — owned by Agentic Operations
  (Module 12 / EPIC-11). This module exposes `search_knowledge()` and `send_notification()`-class
  tool implementations that the Agent Runtime calls under its own policy checks.
- Course content, certification, knowledge-check quiz mechanics — owned by Learning (EPIC-09). This
  module only supplies the `KnowledgeDocument` reference that a `LearningAssignment` can link to.
- Offline sync protocol mechanics, local DB schema, conflict resolution — owned by Mobile Offline
  (Module 04); this module defines *what* is cached (bounded knowledge set, read-receipt queue) not
  *how* sync transport works.
- Analytics dashboards and KPI computation (acknowledgement rate, knowledge search success) — owned
  by Analytics & Control Tower (Module 14); this module emits the source events.

## 3. Personas served

| Persona | Relationship to this module |
|---|---|
| Enterprise Admin | Configures tenant-level comms policy (mandatory channels, escalation defaults, translation provider), knowledge governance policy (review cadence, retirement rules). |
| Operations Admin | Primary author: creates/publishes announcements and knowledge documents, defines audience targeting, reviews audience preview, monitors ack/read completion. |
| Regional/District Manager | Authors announcements scoped to their subtree; reviews non-ack escalations for their team; may author/own knowledge documents scoped to their subtree. |
| Store/Site Manager | Receives and must acknowledge announcements; may be delegated authoring for site-local announcements; consumes Knowledge Hub for operational questions. |
| Frontline Associate | Primary receiver: in-app feed, push, mandatory-ack blocking modal; searches Knowledge Hub; asks AI questions via Operations Copilot. |
| Auditor/Inspector | Consumes knowledge documents (SOPs, standards) during audits; may reference document versions in findings. |
| Executive | Read-only visibility into ack-rate and knowledge-engagement KPIs (via Analytics, not this module directly). |
| Operations Copilot Agent | Consumes this module's authorization-filtered search/RAG API to answer frontline questions; never bypasses document-level authorization. |
| Communication Agent | Produces draft `Announcement` records (status `DRAFT`, `authored_by = agent`) for human review and publish; cannot publish directly (autonomy L1 Recommend per source-spec §20.3). |

## 4. Functional Requirements

### 4.1 Announcement authoring & audience targeting

- **FR-COMMS-001**: The system shall let an authorized author create an `Announcement` with `title`,
  `body` (rich text, max 20,000 chars), `category` (enum: `POLICY`, `SAFETY`, `OPERATIONAL`,
  `HR`, `GENERAL`, tenant-extensible), `priority` (`NORMAL`, `HIGH`, `CRITICAL`), `source_language`,
  and an optional list of attachments (references to Evidence/object-storage assets, max 10 per
  announcement, max 25MB each).
- **FR-COMMS-002**: The system shall support audience targeting by any combination of: (a)
  organization scope — one or more `OrganizationNode` subtrees; (b) role — one or more roles; (c)
  user attribute predicates (e.g. `shift == 'night'`, `certification == 'food_safety'`). Multiple
  scope selections are unioned; role and attribute predicates within a single audience rule are
  intersected (AND) with the scope union. An announcement may carry multiple audience rules, each
  unioned into the final audience set.
- **FR-COMMS-003**: The system shall provide an **audience preview** that resolves and displays the
  estimated recipient count and a sampled list of matching users (name, role, location) before
  publish, computed against the same audience-resolution logic used at publish time (see
  low-level-design.md §6 pseudocode). Preview is non-binding — an org/role/attribute change between
  preview and publish is reflected at publish and again at any resolution refresh (FR-COMMS-015).
- **FR-COMMS-004**: The system shall support marking an announcement `ack_required = true`, with a
  configurable `ack_deadline` (relative, e.g. "within 24h of delivery" or absolute timestamp).
- **FR-COMMS-005**: When `ack_required = true`, the mobile and web in-app experience shall present a
  blocking acknowledgement modal that the user cannot dismiss without acknowledging, before allowing
  continued use of the app's primary work surfaces (frontline "Today" screen), except for
  in-progress safety-critical workflows already open (do not block an active corrective-action
  submission mid-flow — defer the gate to the next natural transition point).
- **FR-COMMS-006**: A non-acknowledgement past `ack_deadline` shall create a `WorkItem` of type
  `Announcement Acknowledgement` in `OVERDUE` state (reusing the universal Work state machine, per
  source-spec §7.1/§17.2) and trigger the configured escalation ladder (notify direct manager at
  T+0, notify manager's manager at T+configurable interval, surface in manager Control Tower
  exceptions list).
- **FR-COMMS-007**: The system shall record a `read receipt` (viewed in feed, opened push, opened
  email) as a distinct, lower-weight signal from `acknowledged` (explicit affirmative action). Both
  are timestamped and immutable once recorded.
- **FR-COMMS-008**: The system shall support scheduled publication (`publish_at` in the future);
  the announcement remains in `SCHEDULED` state and is not visible to any recipient, nor delivered
  on any channel, until the scheduler transitions it to `PUBLISHED`.
- **FR-COMMS-009**: The system shall support `expires_at`; on expiry the announcement is
  automatically removed from the active in-app feed and blocked from further push/email/SMS
  delivery, but remains queryable in history for users who already received it, and remains visible
  in the author's audit/reporting view.
- **FR-COMMS-010**: The system shall support multi-channel delivery per announcement: in-app feed
  (always on), push (opt-out at user preference level, not admin-overridable for `CRITICAL`
  priority), email, and SMS/enterprise-messaging (tenant-level integration toggle, per source-spec
  §11.1). The author selects which channels apply per announcement, subject to tenant policy floors
  (e.g., tenant may mandate push+in-app minimum for `CRITICAL`).
- **FR-COMMS-011**: The system shall support **channel fallback**: if push delivery fails or is not
  acknowledged as delivered within a configurable window (default 15 minutes) for a `CRITICAL`
  announcement, the system shall attempt the next configured channel (e.g., SMS) for that recipient,
  without creating a duplicate in-app feed entry.
- **FR-COMMS-012**: The system shall support a translation workflow: the author designates a
  `source_language`; target languages are either auto-translated (machine translation provider,
  tenant-configurable) or routed to a human translation queue. Each `AnnouncementTranslation` tracks
  its own `status` (`PENDING`, `MACHINE_TRANSLATED`, `HUMAN_REVIEWED`, `STALE`) independent of the
  parent announcement's publish state.
- **FR-COMMS-013**: If the source-language content of a **published** announcement is edited (which
  the system otherwise strongly discourages post-publish — see FR-COMMS-021 immutability rule and
  its correction-only exception), every existing translation for that announcement shall be flagged
  `STALE` and the UI shall display a stale-translation warning banner to any reader viewing that
  translation until it is re-reviewed.
- **FR-COMMS-014**: A recipient's device/locale-preferred language shall determine which translation
  is served by default; if no translation exists in the recipient's preferred language, the system
  falls back to `source_language` content with an in-UI "not available in your language" indicator,
  never a silent fallback.
- **FR-COMMS-015**: Audience membership shall be re-resolved (not just resolved once at publish) on
  a background schedule and on relevant upstream change events (user role change, org reorg, user
  activation/deactivation) for any announcement still within its active window
  (`publish_at <= now < expires_at`), so that new joiners into a targeted scope receive it and users
  who exit scope stop being nagged for ack (see Edge Cases §7 for exact semantics).
- **FR-COMMS-016**: The system shall let an author retract a `PUBLISHED` announcement before its
  natural expiry (`RETRACTED` terminal state), which behaves like expiry (removed from active feed,
  no further delivery/escalation) but is recorded distinctly from `EXPIRED` for reporting.

### 4.2 Knowledge Hub — document lifecycle & scoping

- **FR-KNOW-001**: The system shall model a `KnowledgeDocument` as a stable logical entity (id,
  `owner_user_id`, `category`, `tags[]`) with one or more `KnowledgeDocumentVersion` rows, mirroring
  the `Template`/`TemplateVersion` pattern (source-spec §17.4): `DRAFT → REVIEW → APPROVED →
  PUBLISHED → SUPERSEDED → RETIRED`.
- **FR-KNOW-002**: Only one `KnowledgeDocumentVersion` per document per language may be in
  `PUBLISHED` state at a given point in time; publishing a new version automatically transitions the
  prior published version (same language) to `SUPERSEDED`, retaining it for historical reference
  (e.g., "what did the SOP say when this audit finding was recorded").
- **FR-KNOW-003**: Each `KnowledgeDocumentVersion` shall carry `effective_from` / `effective_to`,
  supporting scheduled and time-bound knowledge (e.g., a seasonal procedure). A version outside its
  effective window is not returned by search or AI-answer even if `PUBLISHED`.
- **FR-KNOW-004**: The system shall support audience scoping per document (and optionally overridden
  per version) using the same scope/role/attribute model as announcements (FR-COMMS-002), stored as
  `KnowledgeDocumentAudience` rules. A document with no audience rule defaults to
  tenant-wide-authenticated-users visibility (least-surprise default; see §9 Resolved Decisions).
- **FR-KNOW-005**: The system shall support multiple language versions of one logical document,
  each independently versioned, reviewed, and published; language versions may drift out of sync
  (see Edge Cases §7 — "multi-language version drift").
- **FR-KNOW-006**: The system shall support full-text and semantic search over `PUBLISHED`,
  effective, audience-authorized `KnowledgeDocumentVersion` content only. Search results shall never
  include `DRAFT`, `REVIEW`, `RETIRED`, or out-of-audience-scope documents for the requesting user,
  regardless of relevance score.
- **FR-KNOW-007**: The system shall provide an AI-answer (RAG) endpoint that retrieves from the same
  authorization-filtered index as direct search (FR-KNOW-006), synthesizes an answer, and returns
  mandatory citations (document id, version id, title, section anchor) for every claim. An answer
  with no authorized supporting passages shall return a "no answer available" response rather than
  an unsourced generative answer.
- **FR-KNOW-008**: Retiring a `KnowledgeDocumentVersion` (`RETIRED`) shall remove it from search and
  from the AI-answer retrieval index (embedding deletion/soft-exclusion) within the indexing SLA
  (§5 NFR), and shall invalidate any cached AI-answer that cited it (see Edge Cases §7).
- **FR-COMMS-021/FR-KNOW-009**: A `PUBLISHED` announcement or `PUBLISHED` document version's
  substantive content is treated as immutable per source-spec Principle P7 (Historical Truth);
  corrections are made by publishing a new version, not silent in-place edits, except for narrowly
  scoped metadata fixes (typo in title, tag correction) that do not change substantive meaning and
  are always audit-logged with a before/after diff.
- **FR-KNOW-010**: Document ownership includes a configurable `review_cadence` (e.g., annual); the
  system shall create a review reminder work item for the owner ahead of the review-due date and
  surface overdue-review documents in an admin dashboard (data feeds Analytics, not rendered by this
  module).

### 4.3 Cross-cutting

- **FR-COMMS-017/FR-KNOW-011**: Every mutating action in this module (publish, retract, ack,
  publish-version, retire) shall emit the corresponding canonical event
  (`message.published`, `message.acknowledged`, and module-scoped equivalents for knowledge —
  `knowledge.document.published`, `knowledge.document.retired` — extending source-spec §18.3) onto
  the Event Backbone in the standard envelope (source-spec §18.4).
- **FR-COMMS-018/FR-KNOW-012**: All list/search/read endpoints shall enforce tenant + organization
  scope + role/attribute authorization server-side; a client-supplied filter can narrow but never
  widen the authorized result set (conventions.md multi-tenancy rule).
- **FR-COMMS-019**: Offline mobile clients shall cache the bounded set of currently-active,
  in-audience announcements and the bounded set of knowledge documents flagged for offline
  availability (tenant/document-level flag, capped by a storage quota) per source-spec §9.2; read
  receipts and acknowledgements captured offline are queued and synced idempotently (client_uuid
  keyed) per source-spec §19.

## 5. Non-Functional Requirements

| Requirement | Target | Notes |
|---|---|---|
| Push delivery latency | p95 < 30s from publish to provider handoff for `CRITICAL`; < 5 min for `NORMAL` | Measured publish-event-to-FCM/APNs-accept |
| Email delivery latency | p95 < 5 min | Via Notification Hub email provider |
| SMS/enterprise-messaging fallback trigger | Within configurable window, default 15 min for `CRITICAL` non-delivery | FR-COMMS-011 |
| In-app feed visibility | < 2s after publish for an already-connected client (via push-triggered refresh or WebSocket/long-poll where available) | Degrades to next foreground sync if disconnected |
| Audience resolution (preview) | p95 < 3s for audiences up to 100k users | Precomputed membership index, not live tree walk (see HLD §7) |
| Fan-out at scale | Support single-announcement fan-out to 1M+ devices within delivery latency targets above | Batched Notification Hub calls, backpressure-aware |
| Knowledge search latency | p95 < 400 ms for lexical query; p95 < 800 ms for semantic/hybrid query | OpenSearch + pgvector hybrid, per conventions.md |
| AI-answer (RAG) latency | p95 < 4s end-to-end (retrieval + generation) | Excludes cold-start model provider latency spikes, which degrade gracefully with a "still working" state |
| Non-ack escalation sweep | Runs at least every 15 min | Background job, §LLD |
| Search index staleness | New/updated published document searchable within 2 min (p95) | Embedding + OpenSearch indexing pipeline |

## 6. UX States

- **Unread badge**: in-app feed icon shows a count of unread + a distinct visual marker for any
  `ack_required` item still unacknowledged (e.g., red dot vs. neutral badge), so mandatory items are
  visually distinguishable from informational ones.
- **Ack-required blocking modal**: full-screen, non-dismissable except via the acknowledgement
  action; shows title, body, translation-language selector (if multiple available), and an explicit
  "I acknowledge" affirmative control (never a passive auto-ack on view). Shows `ack_deadline`
  countdown when present.
- **Knowledge search — empty state**: no results for the query — distinguish "no matching content
  exists" (show a "suggest to admin" / "ask Operations Copilot" affordance) from "results exist but
  none are in your authorized scope" — the latter must **never** reveal that out-of-scope content
  exists (no "3 results hidden" counter); it renders identically to true zero-result.
- **Knowledge search — no permission**: attempting to open a specific document link the user is not
  authorized for returns a generic 403-style "not available" page, not a "you lack permission for
  this specific document" message (avoids confirming document existence to an unauthorized user).
- **Stale-translation warning**: a persistent banner on any translated announcement/document flagged
  `STALE`, with a link to view the current source-language version and a re-review status indicator.
- **Offline-cached knowledge access**: cached documents render with an "available offline · last
  synced <timestamp>" indicator; documents not cached show "requires connection" rather than a
  broken/empty page; searching while offline searches only the local cached bounded set with a
  banner noting results are limited to offline content.
- **Scheduled/expired states in author view**: authors see a distinct badge for `SCHEDULED`,
  `PUBLISHED`, `EXPIRED`, `RETRACTED` announcements and `DRAFT`/`REVIEW`/`PUBLISHED`/`SUPERSEDED`/
  `RETIRED` document versions in their authoring list, with an audit trail link on each.
- **Loading/error states**: standard skeleton loaders for feed/search; network-error state on
  search/ask distinguishes "connectivity problem — retry" from "no results" (never conflate them).

## 7. Validation & Edge Cases

- **Overlapping audience scopes with conflicting announcements**: a user may fall into multiple
  audience rules across different announcements that are not logically conflicting (each
  announcement is independent) — the system does not attempt cross-announcement conflict detection
  by content; it is an author/governance responsibility. The system does deduplicate: a user
  matching multiple audience *rules within the same announcement* still receives exactly one
  delivery per channel per announcement (audience resolution produces a de-duplicated recipient
  set before fan-out).
- **Ack required but user deactivated mid-campaign**: if a targeted user is deactivated
  (`User.status != ACTIVE`) before acknowledging, the pending `Announcement Acknowledgement`
  WorkItem is transitioned to `CANCELLED` (not `OVERDUE`/escalated) on the next audience
  re-resolution cycle (FR-COMMS-015); no escalation notification is sent for a deactivated user.
  If the user is later reactivated before `expires_at` and still matches the audience, the
  acknowledgement requirement is re-instated with a fresh deadline computed from reactivation time
  (not the original deadline, which may already have passed).
- **Knowledge doc referenced by AI answer after being retired**: the AI-answer endpoint always
  retrieves live against the current authorized/effective index at answer time — it does not serve
  cached answers that cite a since-retired version without re-validating. Any previously **cached**
  answer (see LLD caching strategy) that cites a document version transitioning to `RETIRED` is
  proactively invalidated by the retirement event handler; a client re-asking gets a fresh answer
  (or "no longer available" if no replacement exists) rather than a stale citation to a retired
  source. Historical audit/citation records already delivered to a user are not retroactively
  altered (P7 Historical Truth) but are visually flagged if re-displayed after the fact.
- **Multi-language document version drift**: language versions of the same logical document are
  independently versioned and may legitimately be at different version numbers (e.g., EN v4 while
  ES is still v3 pending translation). The system surfaces a "translation may be outdated"
  indicator whenever the non-source-language version's `source_version_reference` is older than the
  current published source-language version, without blocking access to the older translation
  (blocking would leave non-English-primary users with no content at all, which is worse).
- **Audience preview vs. actual send drift**: because audience membership can change between preview
  and publish (org change, new hire), the preview is explicitly labeled as an estimate; the
  authoritative recipient list is the one resolved and snapshotted at the moment of `PUBLISHED`
  transition (`AnnouncementTarget` rows), not the preview.
- **Channel fallback loop prevention**: a recipient with no valid contact method on the fallback
  channel (e.g., no phone number for SMS fallback) is recorded as `DELIVERY_FAILED` with reason
  `NO_CONTACT_METHOD` rather than retried indefinitely.
- **Duplicate acknowledgement**: acknowledging an already-acknowledged announcement (e.g., double-tap,
  offline queue replay) is idempotent — the first `AnnouncementAck` row wins; subsequent attempts
  return 200 with the original ack timestamp, no duplicate row (mirrors source-spec §19.4 submission
  idempotency rule).
- **Knowledge document ownership transfer**: transferring `owner_user_id` (e.g., owner leaves the
  company) is an audited action; a document cannot be left ownerless — transfer must specify a new
  owner in the same request, or the system defaults ownership to the requesting admin with a
  flagged "needs reassignment" state.
- **Translation of a retracted/expired source**: no new translation work is queued against an
  announcement that is no longer `PUBLISHED`/`SCHEDULED`; in-flight human translation tasks against
  a retracted announcement are auto-cancelled.

## 8. Dependencies

| Dependency | Direction | Nature |
|---|---|---|
| Identity & Organization (Module 01) | Consumes | `OrganizationNode` tree, `UserScope`, `Role`, `User` status for audience targeting/resolution and search authorization filters. |
| Mobile & Offline (Module 04) | Provides to / Consumes from | Supplies bounded offline-cache manifest (active announcements, flagged knowledge docs); consumes offline-queued read receipts/acks for idempotent sync. |
| Agentic Operations (Module 12 / EPIC-11) | Provides to | Exposes `search_knowledge()` tool and RAG endpoint to the Operations Copilot; consumes draft `Announcement` records produced by the Communication Agent for human review. |
| Learning (EPIC-09) | Provides to | `KnowledgeDocument` reference for course content linkage and knowledge-check source material. |
| Notification Hub (platform-shared) | Provides to | Delivery requests (push/email/SMS) with dedupe keys; consumes delivery-status callbacks. |
| Analytics & Control Tower (Module 14) | Provides to | `message.published`/`message.acknowledged`/knowledge events feed ack-rate and knowledge-search-success KPIs. |
| Evidence (Module 05) | Consumes | Attachment storage for announcement media and document embedded assets uses the shared Evidence/object-storage pattern (hashing, signed URLs). |

## 9. Resolved Decisions

- **Default audience for a `KnowledgeDocument` with no explicit audience rule (FR-KNOW-004)**:
  tenant-wide, all authenticated active users. This is firm, not merely recommended — an
  accidentally unscoped operational SOP being invisible to everyone is a worse failure mode than
  being visible to slightly too many internal employees. Tenant admins may override this default
  tenant-wide via a tenant policy setting; the per-document scoping rule (FR-KNOW-004) always wins
  once explicitly set.
- **Machine translation provider (FR-COMMS/FR-KNOW translation flows)**: MT runs behind a
  provider-agnostic interface (analogous to the Agent Runtime model gateway, conventions.md), with a
  single managed MT provider (DeepL Pro API) as the platform default at GA — one vendor keeps the
  translation-quality/glossary-management surface simple for launch, and the abstraction layer
  means switching or adding a second provider later is a config change, not a rearchitecture.
  Tenants may additionally require human-review-before-publish for machine-translated content in the
  `SAFETY` category (and any other tenant-designated regulated category); this human-review gate is
  tenant-configurable and defaults to **off** for non-`SAFETY` categories, **on** for `SAFETY`.
- **Enterprise messaging integration (Teams/Slack-class, source-spec §11.1)**: implemented as an
  additional one-way delivery channel behind the Notification Hub abstraction, alongside push/email/
  SMS — not a bidirectional collaboration surface. There is no reply-in-Slack/Teams acknowledgement;
  acknowledgement must happen in-app/mobile so there is exactly one authoritative ack record per
  announcement. Channel-native acknowledgement is out of scope for this module and would require a
  new epic if a tenant demand emerges.
- **AI-answer caching**: permitted only when the cache key includes the requesting user's resolved
  authorization fingerprint (not just the question text), so a cached answer is never served across
  differing authorization contexts. See LLD §8 caching strategy for the fingerprint composition.
- **`CRITICAL`-priority announcements and push opt-out (FR-COMMS-010)**: push delivery is always
  attempted by the platform for `CRITICAL` announcements — this channel-attempt behavior is not
  admin-overridable at the tenant level, at any tenant. This never bypasses the OS-level user
  notification permission itself (the platform cannot and does not attempt to circumvent a
  device-level opt-out; that is outside the platform's control). Where a tenant's jurisdiction
  restricts compelled workplace notifications, the restriction is enforced through the
  Security/Compliance module's (13) tenant compliance profile, which can require the in-app feed and
  ack-blocking modal (both platform-controlled surfaces the tenant fully governs) to serve as the
  guaranteed-delivery path instead of push, rather than through a per-announcement or
  per-Communications-module toggle — keeping the jurisdiction-sensitive control in one place
  (module 13) instead of duplicated per delivery channel.
