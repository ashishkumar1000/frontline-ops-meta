# Module 07 — Communications & Knowledge

> This module gets the right information to the right frontline people and can prove it arrived: announcements with mandatory, tracked acknowledgement, plus a searchable library of approved operating knowledge (SOPs, standards). Store teams must explicitly confirm critical updates like a new safety policy, and everyone — including the AI copilot — answers operational questions only from documents they are allowed to see.

| | |
|---|---|
| **Module code** | `07-communications-knowledge` |
| **Owning service** | CommunicationsService + KnowledgeService (two deployables, one bounded context) |
| **Epics** | EPIC-08 (Comms & Knowledge) |

## In plain terms
- Publish an announcement to a precise audience — an org subtree, a role, or an attribute like "night shift" — and preview exactly who will receive it before sending.
- Make an announcement mandatory: the app shows a blocking "I acknowledge" screen until the deadline; people who don't acknowledge generate an overdue work item that escalates to their manager.
- Reach people wherever they are: in-app feed, push, email, SMS or Teams/Slack-style channels, with automatic fallback if a critical message is not delivered; announcements can be scheduled, expired, or retracted.
- Translate announcements into each reader's language, and flag translations as stale when the source text changes.
- Author knowledge documents with versioning, effective dates, review cadences, and independent per-language versions.
- Search everything with keyword + semantic search, or ask a question and get an AI answer built only from documents you are authorized to see — always with citations, never a guess.

## Key concepts
| Concept | What it means |
|---|---|
| Announcement | A time-bound message (priority `NORMAL`/`HIGH`/`CRITICAL`) delivered to a resolved audience; published content is immutable — corrections mean a new version or a retraction. |
| Audience rule & AudienceResolver | The shared component that resolves "who can see this" from org scope + role + user attributes; used identically by the comms and knowledge halves. |
| Acknowledgement vs. read receipt | An explicit "I confirm" versus a passive "seen"; both are timestamped and immutable once recorded. |
| Announcement Acknowledgement WorkItem | What a missed acknowledgement deadline becomes — an overdue task running the standard escalation ladder. |
| KnowledgeDocument / KnowledgeDocumentVersion | One logical document (e.g. an SOP) with immutable per-language versions moving through `DRAFT → REVIEW → APPROVED → PUBLISHED → SUPERSEDED → RETIRED`. |
| AI-answer (RAG) endpoint | Answers questions by retrieving only from authorized, in-effect documents and citing them; with no authorized source it returns "no answer available" instead of an unsourced answer. |
| Channel fallback | If a `CRITICAL` push is not delivered within 15 minutes, the system tries the next configured channel (e.g. SMS) for that recipient. |
| Stale translation | A translation flagged when its source-language content changes after publish, shown with a warning banner until re-reviewed. |

## How it connects to the rest
- Consumes **Identity & Organization (01)**: the org tree, roles, scopes, and user attributes are what audience targeting and search authorization are resolved against.
- Delivers through the platform **Notification Hub**; this module owns the business record (who was targeted, who acknowledged), not the transport itself.
- Serves **Agentic Ops (10)**: the Operations Copilot answers questions via this module's `search_knowledge()`/RAG API, and the Communication Agent drafts announcements that a human must review and publish.
- Feeds **Learning (08)** (courses can link to knowledge documents) and **Mobile Offline (04)** (a bounded offline cache of active announcements and flagged documents); emits `message.*` and `knowledge.*` events consumed by **Analytics (11)** for ack-rate and search-success KPIs.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-COMMS-001–021 and FR-KNOW-001–012: authoring, audience targeting, acknowledgement and escalation, translation, document lifecycle, authorized search and RAG, plus delivery-latency NFRs and UX states. |
| [high-level-design.md](./high-level-design.md) | Why one bounded context ships as two deployables (CommunicationsService, KnowledgeService) sharing one AudienceResolver; component diagram, data ownership, million-device fan-out, per-tenant search isolation, failure modes, and trade-offs. |
| [low-level-design.md](./low-level-design.md) | Postgres schema (announcements, targets, translations, acks, documents, versions, embeddings), REST endpoints, publish/search/RAG sequence diagrams, audience-resolution and authorization-filtered query pseudocode, caching, and background jobs. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: component view, announcement fan-out sequence, authorized knowledge search/RAG sequence, the entity-relationship diagram, and the announcement lifecycle state machine. |
