# Module 10 — Agentic Operations (Agent Runtime)

> This module is the platform's governed AI layer: instead of a human having to notice a problem and act, AI agents watch operational events, reason about them, and either act automatically or ask a human for approval — always inside strict policy boundaries. It solves the trust problem of AI in operations: every agent action passes through one fixed, auditable pipeline, and a human approves anything that is not proven safe. It is used by managers and admins (who approve and configure agents), and by frontline staff (who receive agent-assigned tasks and answers).

| | |
|---|---|
| **Module code** | `10-agentic-operations` |
| **Owning service** | Agent Runtime Service (single logical service; all 9 agents share one runtime, one pipeline, one policy engine) |
| **Epics** | EPIC-11 Agent Runtime |

## In plain terms
- Nine purpose-built AI agents each do one narrow job: spot repeated failures (Compliance Monitor), answer staff questions with sources (Operations Copilot), propose fixes for issues (Resolution Agent), assign micro-learning (Training Agent), draft announcements (Communication Agent), summarize audits (Audit Intelligence Agent), interpret photo-inspection results (Visual Inspection Agent), explain executive KPIs (Executive Insights Agent), and draft workflows from plain-language descriptions (Workflow Builder Agent).
- Managers review pending agent actions in one place — the **Agent Inbox** — and can approve, reject, modify, or ask for more evidence.
- Admins set per-agent rules: which tools an agent may use, which actions always need human approval, and a **kill switch** that stops an agent instantly at tenant, agent, or tool level.
- Every agent run is recorded end-to-end (what it saw, what policy allowed, what tools it called, what the human decided) in an immutable audit trail.
- Autonomy levels (L0 observe-only up to L4 prohibited) control how much an agent can do without a human; low-confidence or high-severity actions always route to a human.
- Guardrails are deterministic: a rules engine (OPA/Rego — a tool that evaluates policies as code) decides what an agent may do, never the AI model itself, and no AI output can silently override an enterprise policy.

## Key concepts
| Concept | What it means |
|---|---|
| **Agent Loop** | The fixed 12-stage pipeline every agent run follows (event, context, policy check, model call, retrieval, reasoning, tool selection, authorization, execution, validation, audit, human escalation). No agent can skip or reorder the policy or audit stages. |
| **Autonomy levels (L0–L4)** | How much an agent may do alone: L0 observe only, L1 recommend, L2 auto-execute safe actions, L3 requires human approval, L4 blocked entirely. |
| **Agent Inbox** | The single queue of pending agent actions awaiting a human approve/reject/modify decision, shown to the right manager by scope. |
| **AgentPolicy** | A versioned per-tenant, per-agent configuration (allowed tools, blocked tools, actions requiring approval) compiled to OPA/Rego rules. |
| **AgentRun / AgentToolExecution** | The immutable audit records for a run and for each tool call inside it — what happened, under which policy version, approved by whom. |
| **Kill switch** | An Enterprise Admin's fail-closed emergency stop for any agent, at tenant, agent, or tool level; a disabled agent stops before any model call. |
| **Typed tool catalog** | The fixed allowlist of actions an agent can take — each is a normal platform API call (e.g. `create_task`, `create_issue`) with a fixed schema; agents never get free-form code execution or a private backdoor into other modules. |
| **RAG (retrieval-augmented generation)** | How agents ground answers: they first search tenant-authorized knowledge and history, cite the sources, and refuse to answer when no good source exists. |

## How it connects to the rest
- It is a client of nearly every module's APIs, by design: agents call the same governed APIs a human UI would call — Work Orchestration (02), Issues/CAPA (06), Communications/Knowledge (07), Learning (08), Visual AI (09, read-only), Evidence (05, read-only) — and never hold their own copy of business data.
- It consumes domain events from those same modules as triggers (e.g. `task.overdue`, `issue.created`, `audit.submitted`), so nearly any operational event can start an agent run.
- It depends on Identity/Organization (01) for every scope decision — the Context Builder assembles only data the agent is authorized to see, enforced structurally, and cross-tenant retrieval is impossible.
- It emits `agent.*` events consumed by Analytics & Control Tower (11), which powers the Automation Rate and AI Override Rate dashboards — the one true bidirectional relationship (Analytics' metrics API is also the Executive Insights Agent's data source).
- Its outputs feed module 03's lifecycle: the Workflow Builder Agent only ever produces a DRAFT form/workflow; publishing is a human-only action, permanently.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | FR-AGENT-0xx/1xx/2xx/3xx/4xx requirements: the 12-stage Agent Loop contract, behavior of all 9 agents (FR-AGENT-101–109), the Agent Inbox, kill switch, plus binding EU AI Act controls (FR-AGENT-500–503). |
| [high-level-design.md](./high-level-design.md) | The Agent Runtime Service's five responsibilities (Context Builder, Policy Engine, Model Gateway, Tool Executor, Audit/Evaluation), data ownership, scaling, multi-tenancy, and fail-closed degradation strategy. |
| [low-level-design.md](./low-level-design.md) | Database schema (AgentRun, AgentPolicy, ApprovalRequest, etc.), the typed tool catalog, REST endpoints, sequence diagrams for L2/L3 flows, and the autonomy-decision pseudocode. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: component/container view, L2 auto-execute and L3 approval sequences, entity-relationship diagram, and the autonomy-level decision flow. |
