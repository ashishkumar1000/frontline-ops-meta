# Agentic Operations (Agent Runtime) — Functional Specification

Module: `10-agentic-operations` | Source: source-spec.md §13 (Agentic Operations), §20 (AI Runtime
and Governance FSD), Appendix C (Example Agent Policy), Appendix E EPIC-11 | Epic: **EPIC-11 Agent
Runtime**

## 1. Purpose and Scope

### 1.1 Purpose
The Agent Runtime is the platform's governed AI execution layer. It is the mechanism by which the
platform moves work from "a human must notice and act" to "the system observes, reasons, and acts
within policy boundaries, with a human in the loop wherever the action is not proven safe." It is
the durable differentiator called out in source-spec §1 ("Operational Graph... + governed agents")
and the PRD outcome "Create a safe path from rules-based automation to governed autonomous agents."

The Agent Runtime is a platform capability, not a single feature screen. It:
- Hosts a portfolio of 9 purpose-built agents (§2 below), each with a narrow job, not one general
  chat agent with unlimited scope.
- Runs every agent invocation through one deterministic pipeline — the **Agent Loop**
  (§13.2 of source-spec, deepened in §3 below) — so behavior, audit, and governance are uniform
  regardless of which agent or which LLM provider is behind it.
- Enforces autonomy levels (L0–L4, per conventions.md, reused verbatim) and a per-agent, per-tenant
  policy (allowed/blocked tools, approval-required conditions) evaluated by OPA/Rego, never by the
  LLM itself.
- Exposes every pending human decision through a single, consistent **Agent Inbox** experience,
  and every executed action through an immutable audit trail.
- Acts as a **typed tool-calling client of nearly every other module** — it does not have its own
  copy of business data or its own write path into domain objects; it calls the same governed APIs
  that a human-driven UI would call (Issues/CAPA, Work Orchestration, Communications, Learning,
  Visual AI, Knowledge), through an allowlisted tool catalog.

### 1.2 In Scope
- The 9-agent portfolio: functional behavior, triggers, inputs/outputs, default autonomy, escalation.
- The Agent Loop pipeline stages as a functional contract (what each stage must guarantee).
- The deterministic-rule + AI pattern (source-spec §13.3): which decisions must be rule-based vs.
  model-based, and the non-negotiable rule that no LLM output silently overrides policy.
- Human-in-the-loop approval: the Agent Inbox UX, approval card contents, approve/reject/modify/
  request-more-evidence actions.
- Kill-switch UX and scope (tenant/agent/tool level).
- Agent policy configuration UX for admins (allowed_tools, blocked_tools, approval_required_for).
- Agent Run Record and Agent Evaluation metrics as functional (not just technical) requirements —
  they drive the Agent Governance surface an Enterprise Admin uses to trust or distrust an agent.
- Tool catalog *functional contract* (full typed schema lives in low-level-design.md).

### 1.3 Out of Scope (owned by other modules; this module calls them, does not define them)
- The business logic, state machines, and UI of Work Orchestration (module 02), Forms/Workflow
  Builder (module 03), Evidence (module 05), Issues/CAPA (module 06), Communications/Knowledge
  (module 07), Learning (module 08), Visual AI (module 09) — this module's agents call their APIs
  as tools; this spec defines the *tool contract*, not the target module's own object lifecycle.
  Note: for the Workflow Builder Agent specifically, "workflow" refers to the no-code
  form/workflow definitions owned by module 03 — this module only produces a *draft* artifact that
  module 03's DRAFT → REVIEW → APPROVED → PUBLISHED lifecycle then owns.
- Model provider selection/billing infrastructure, embedding pipeline internals, and the
  Knowledge Hub's own document ingestion — this module only calls `search_knowledge()`.
- Org/RBAC scope evaluation mechanics — this module consumes UserScope/OrganizationNode data via
  the Context Builder and defers to the Authorization Service as the source of truth.
- Analytics/Control Tower dashboards (module beyond this set) that visualize AI Override Rate,
  Automation Rate — this module emits the underlying events and metrics; the dashboard is owned
  elsewhere.

## 2. Personas

### 2.1 Human Personas (as approvers/reviewers/configurers of agent output)

| Persona | Role with respect to Agent Runtime |
|---|---|
| Enterprise Admin | Configures tenant-wide agent policy, model/provider allowlists, kill switches, autonomy-level ceilings per agent; reviews Agent Evaluation dashboard; is the only role that can raise an agent's autonomy level above the platform default or approve L3 tools tenant-wide. |
| Operations Admin | Configures agent behavior scoped to their standards/workflows (e.g., which Compliance Monitor rule sets feed which locations); reviews and tunes escalation thresholds within limits set by Enterprise Admin. |
| Regional/District Manager | Primary approver for L3 actions and reviewer of L1 recommendations affecting their territory in the Agent Inbox; sees agent-driven exceptions in their Control Tower. |
| Store/Site Manager | Approver for L3 actions scoped to their location; recipient of agent-drafted communications and agent-created tasks/issues; can reject/modify agent recommendations for their site. |
| Frontline Associate | Recipient of agent-assigned tasks, agent-drafted micro-learning, agent-drafted notifications; generally not an approver, but can flag an agent action as incorrect (feeds override/evaluation metrics). |
| Auditor/Inspector | Consumer of Audit Intelligence Agent summaries; reviewer/approver of Visual Inspection Agent findings that require human confirmation. |
| Executive | Consumer of Executive Insights Agent output; approver of enterprise-wide policy changes that affect agent autonomy ceilings (governance sign-off, not day-to-day approval). |
| AI Agent (system persona, one per agent type) | See §4 — each of the 9 agents is itself a system persona with a defined identity (`agent_id`, `agent_version`), scope of authorized context, and tool allowlist. Agents never act as a human principal; every action they take is attributable to the agent identity plus the policy version that authorized it. |

### 2.2 The "AI Agent" System Personas
Each of the 9 agents is modeled as a distinct system persona for authorization, audit, and policy
purposes — never a shared, undifferentiated "AI" identity:
`compliance_monitor`, `operations_copilot`, `resolution_agent`, `training_agent`,
`communication_agent`, `audit_intelligence_agent`, `visual_inspection_agent`,
`executive_insights_agent`, `workflow_builder_agent`.

## 3. The Agent Loop — Functional Contract

Per source-spec §13.2, every agent invocation, regardless of agent type, passes through the same
eleven stages. This section defines what each stage must functionally guarantee (technical
implementation in high-level-design.md / low-level-design.md).

| # | Stage | Functional guarantee |
|---|---|---|
| 1 | **Event** | A canonical domain event (source-spec §18.3, e.g. `issue.created`, `audit.submitted`) or a scheduled/manual trigger initiates a run. Every run has exactly one triggering cause, recorded verbatim. |
| 2 | **Context Builder** | Assembles *only* context the invoking agent is authorized to see: tenant-scoped, org-node-scoped (per the triggering event's location/work item), time-bounded (no unbounded history unless the agent's contract explicitly allows it, e.g. Compliance Monitor's `get_recent_failures`). Cross-tenant retrieval is structurally impossible (FR-AGENT-041). |
| 3 | **Policy Check (pre)** | Before any model call, the agent's tenant policy (Appendix C style) is evaluated: is this agent enabled for this tenant/location? Is the triggering condition within the agent's configured scope? A disabled or kill-switched agent stops here — no model call is made, no cost incurred. |
| 4 | **Agent / Model** | The provider-agnostic model gateway is invoked with the agent's versioned prompt/contract. Every model call is recorded (provider, model id, prompt_version, cost, latency). |
| 5 | **Retrieval** | RAG lookups against Knowledge Hub / operational history via `search_knowledge()` and similar tools happen under the same authorization/tenant boundary as stage 2. Retrieved source IDs are recorded (`retrieved_sources[]`) for citation and audit. |
| 6 | **Reasoning** | The model produces a structured (not free-text-only) recommendation: decision, rationale, evidence references, confidence score. Free-text-only output that cannot be parsed into the structured contract is treated as a failure (§7 edge cases). |
| 7 | **Tool Selection** | The model proposes zero or more tool calls from its allowlisted catalog only; it cannot propose an arbitrary action or arbitrary code. A proposed tool outside the allowlist is rejected at this stage and logged, not silently dropped. |
| 8 | **Action Authorization** | The autonomy-level decision function (pseudocode in low-level-design.md) determines, per proposed tool call: execute automatically (L2), require human approval (L3), or block entirely (L4/policy `blocked_tools`/`approval_required_for` match). This is a **deterministic** policy evaluation (OPA/Rego), never delegated to the LLM. |
| 9 | **Tool Execution** | Authorized tool calls are executed against the target module's real write API, with the same idempotency-key discipline as any other mutating call (conventions.md API conventions). Execution is attributed to the agent identity + `AgentRun.id` for downstream audit. |
| 10 | **Result Validation** | The tool's response is validated against its declared return schema; a mismatch or tool-level error triggers the failure/compensating-action path (§7). |
| 11 | **Audit Log** | Every stage's material facts (context hash, policy decision, tools called, arguments, results, human approval/rejection) are written to the immutable `AgentRun`/`AgentToolExecution` record (source-spec §20.4) before the run is considered complete. |
| 12 | **Human Escalation / Continue** | If any action required approval (L3) or the agent's own confidence is below the tenant-configured threshold, the run parks in the Agent Inbox (§6) awaiting a human decision; otherwise the run closes as `COMPLETED`. |

This module treats stages 1–12 as a **fixed pipeline contract** — any of the 9 agents may skip
stages that don't apply to it (e.g., an L0 Observe-only agent never reaches stage 8/9), but no agent
may reorder or bypass the Policy Check or Audit Log stages.

## 4. Per-Agent Functional Behavior

For each agent: trigger conditions, inputs, expected outputs, default autonomy level (per
source-spec §13.1, all default to **L1 Recommend** except where noted — Appendix C shows
`compliance_monitor` configured up to L2 as a tenant-level policy override, not the platform
default), and escalation conditions.

### 4.1 FR-AGENT-101 — Compliance Monitor
- **Trigger conditions:** Canonical events indicating a potential deviation — `audit.submitted`
  (with a rule-flagged risk field), `task.overdue` beyond SLA multiplier, repeated
  `issue.created` events for the same location/checklist-item within a rolling window (default 3
  occurrences / 30 days, tenant-configurable), and scheduled sweep triggers (e.g., nightly
  cross-location pattern scan).
- **Inputs:** Triggering event payload; `get_recent_failures(location_id, item_id, window)`;
  `get_active_policy(tenant_id, scope)`; `get_location(location_id)` for hierarchy/rollup context.
- **Expected outputs:** A structured finding: `{pattern_type, affected_locations[], severity,
  confidence, recommended_action, evidence_refs[]}`. Recommended actions are proposals for the
  Resolution Agent or direct tool calls (`create_issue`, `create_task`) depending on configured
  autonomy.
- **Default autonomy:** L1 Recommend (platform default); tenants may raise to **L2** for
  `create_task`/`create_issue`/`send_notification` per Appendix C, never above L2 without an
  `approval_required_for` override for CRITICAL severity.
- **Escalation conditions:** `severity=CRITICAL` always routes to Agent Inbox regardless of
  configured autonomy (matches `approval_required_for` in Appendix C); confidence below tenant
  threshold (default 0.75) routes to Inbox as "low-confidence flagged for review" (§6.4).

### 4.2 FR-AGENT-102 — Operations Copilot
- **Trigger conditions:** Direct user query from mobile/web ("ask a question" surface); no
  autonomous trigger — this agent is invocation-only, never event-driven.
- **Inputs:** User's natural-language question; user's authorization scope (which
  KnowledgeDocuments/WorkItems/locations they may see); conversation context (bounded to the
  current session, no persistent cross-session memory by default — see Agent Contract memory
  policy, §20.1 source-spec).
- **Expected outputs:** A grounded answer with `source_citations[]` referencing specific
  KnowledgeDocument versions and/or operational data (e.g., "your last 3 audits at Store #204");
  `confidence`; and, when the answer implies an action ("should I create a task for this?"), a
  proposed tool call routed through the same Action Authorization stage as any other agent.
- **Default autonomy:** L0 Observe for answering (pure retrieval+synthesis is not a "tool action"
  requiring approval); any follow-on action the user asks it to take (e.g., "assign this to me")
  is evaluated at L1/L2 per the target tool's own minimum autonomy (tool catalog, low-level-design.md).
- **Escalation conditions:** Cannot find an authorized source above a minimum relevance threshold →
  must respond "no confident answer found," never fabricate; must never answer using
  cross-tenant or out-of-scope KnowledgeDocuments (hard block, not a policy the model can reason
  around).

### 4.3 FR-AGENT-103 — Resolution Agent
- **Trigger conditions:** `issue.created`, `issue.escalated`, or a referral from Compliance
  Monitor's finding; also invocable manually by a manager from an open Issue.
- **Inputs:** Issue record (severity, location, category, prior CAPA history via
  `get_recent_failures`); `get_active_policy`; knowledge of prior successful interventions for
  similar issue signatures (retrieval stage).
- **Expected outputs:** A structured corrective-action proposal: `{root_cause_hypothesis,
  recommended_capa_steps[], recommended_owner, recommended_due_date, confidence,
  proposed_tool_calls[]}` — typically `create_task` (corrective action work item),
  `schedule_reinspection`, and optionally `schedule_training` (handed to Training Agent) or
  `request_approval` for high-severity cases.
- **Default autonomy:** L1 Recommend. Tenants may raise routine, low-severity corrective task
  creation to L2; root-cause-driven CAPA plan changes and any action tagged `severity=CRITICAL`
  remain L3 minimum.
- **Escalation conditions:** Proposed action would modify an Issue already being acted on by
  another actor (human or agent) — see §7.2 conflicting-proposals edge case; proposed corrective
  action exceeds a cost/impact threshold (tenant-configurable) → forced L3.

### 4.4 FR-AGENT-104 — Training Agent
- **Trigger conditions:** Referral from Resolution Agent or Compliance Monitor when a failure
  pattern maps to a knowledge/skill gap (per source-spec §11.3 REPEATED FAILURE -> IDENTIFY GAP ->
  ASSIGN MICRO-LEARNING -> KNOWLEDGE CHECK -> RE-INSPECTION -> MEASURE IMPROVEMENT); also
  schedulable on a recurring basis for proactive gap analysis.
- **Inputs:** Failure/issue signature; affected user(s)/role(s); existing Course/LearningAssignment
  catalog (read-only lookup into module 08); user's completed-training history.
- **Expected outputs:** `{recommended_course_id, target_users[]/target_role, due_date,
  requires_knowledge_check, linked_reinspection_work_item_id, confidence}`; tool calls:
  `schedule_training`, `schedule_reinspection`.
- **Default autonomy:** L1 Recommend for course selection targeting named individuals; **L2** is
  the common tenant configuration for role-based, non-disciplinary micro-learning assignment
  (low blast radius); assigning training that is tied to a formal performance/disciplinary record
  is always L3.
- **Escalation conditions:** No matching course exists in the catalog above a relevance threshold →
  escalate to Operations Admin as a "content gap" finding rather than assigning a poor-fit course;
  target user already has an open assignment for the same competency → do not duplicate, escalate
  only if the open assignment is overdue.

### 4.5 FR-AGENT-105 — Communication Agent
- **Trigger conditions:** Referral from any other agent needing to notify humans in structured,
  audience-targeted language (e.g., Resolution Agent needs a manager notified); direct request from
  an Operations Admin ("draft an announcement about X"); scheduled campaign-related triggers.
- **Inputs:** Target audience/scope; source facts (issue, policy change, campaign); tenant tone/
  brand/localization settings; translation requirements (module 07 capability).
- **Expected outputs:** A **draft** announcement/notification (`{subject, body, audience_scope,
  channels[], requires_acknowledgement, expiry, translations{}}`) — this agent never
  auto-publishes; publication is always at minimum L2 with tenant policy typically requiring L3 for
  `external_customer_notification=true` (per Appendix C `approval_required_for`) and for any
  enterprise-wide (top-of-hierarchy scope) announcement.
- **Default autonomy:** L1 Recommend (draft-only) is the floor; `send_notification` execution for
  narrow, internal, low-severity audiences may be configured to L2; anything customer-facing or
  enterprise-wide is L3 minimum, and is L4-prohibited for tenants that disable AI-authored
  external communication entirely.
- **Escalation conditions:** Draft references a policy or fact the agent's confidence in is below
  threshold → flag rather than send; audience resolves to more than a tenant-configured recipient
  ceiling (e.g., >500 recipients) → forced L3 regardless of content.

### 4.6 FR-AGENT-106 — Audit Intelligence Agent
- **Trigger conditions:** `audit.submitted`, `audit.approved`, or scheduled rollup (e.g., end of
  audit cycle); manual invocation from Audit Workspace ("summarize this audit program").
- **Inputs:** One or more Submission records + their TemplateVersion; prior-period submissions for
  trend comparison; related Issues.
- **Expected outputs:** Narrative + structured summary: `{key_findings[], trend_deltas[],
  repeat_pattern_flags[], recommended_focus_areas[], confidence}`; strictly read/summarize — this
  agent's tool allowlist contains no write tools by default.
- **Default autonomy:** L0 Observe (this agent produces insight only; it has no default write
  tools). A tenant may optionally grant it `create_issue` at L1/L2 if it detects a pattern an audit
  reviewer confirms is issue-worthy, but this is an explicit opt-in, not the default contract.
- **Escalation conditions:** None inherent to L0 (nothing to approve); if opted into write tools,
  same escalation rules as Compliance Monitor apply to those tools.

### 4.7 FR-AGENT-107 — Visual Inspection Agent
- **Trigger conditions:** New Visual AI result available (module 09 emits an internal event once
  its model pipeline scores an image) where the result contract's `confidence` is below the
  auto-accept threshold, or where `status=AI_RESULT_UNREVIEWED` and the use case requires narrative
  interpretation (e.g., translating bounding-box violations into a corrective-action recommendation
  a human can act on quickly).
- **Inputs:** Visual AI result contract (source-spec §12.2: `compliance_score, confidence, status,
  violations[]`); the reference planogram/standard; location/asset context.
- **Expected outputs:** Human-readable interpretation of the visual finding + a recommended action
  (`create_issue` for confirmed violations, `request_approval`/flag-for-review for ambiguous ones);
  this agent does not alter the underlying Visual AI result or its `status` — that transition is
  owned by module 09's own review workflow; it only recommends downstream operational action.
- **Default autonomy:** L1 Recommend. `create_issue` for a HIGH-severity, high-confidence
  (≥ tenant threshold, e.g. 0.9) violation may be configured to L2; anything below the visual
  confidence threshold is always routed to human review, never auto-actioned (mirrors module 09's
  own "low-confidence enters human-review state" acceptance test, source-spec §24.1).
- **Escalation conditions:** `confidence` below the visual-AI module's own review threshold (always
  escalate, this agent cannot override module 09's threshold); violation severity CRITICAL (e.g.,
  safety/PPE) always forces L3 regardless of confidence.

### 4.8 FR-AGENT-108 — Executive Insights Agent
- **Trigger conditions:** Scheduled (e.g., weekly enterprise digest), or on-demand from Executive
  Control Tower ("explain this KPI shift").
- **Inputs:** Aggregated analytics facts (Verified Operational Execution Rate and other KPIs from
  source-spec §14.2), cross-location/region rollups, prior-period baselines; this agent reads from
  the analytics/warehouse layer, not raw transactional tables, to avoid heavy transactional load.
- **Expected outputs:** Narrative explanation of *what changed, where, why, and what needs
  attention* mapped directly to the Executive Questions in source-spec §14.1, with drill-down
  evidence references; no default write tools (L0).
- **Default autonomy:** L0 Observe, always. This agent never executes or proposes tool actions by
  default; it may recommend that a human "review the Compliance Monitor findings for Region X,"
  which is a pointer, not an action.
- **Escalation conditions:** N/A (no actions to escalate); a "confidence" concept still applies to
  its causal narrative (e.g., correlation vs. asserted causation) and must be surfaced, not hidden.

### 4.9 FR-AGENT-109 — Workflow Builder Agent
- **Trigger conditions:** Manual invocation only — an Operations Admin describes a desired
  workflow/form in natural language from the Workflow Builder UI (module 03).
- **Inputs:** Natural-language description; existing template/component library (for reuse);
  organization's field/lookup reference data (for lookup-field suggestions).
- **Expected outputs:** A **draft** TemplateVersion in `DRAFT` state (module 03's own state
  machine) — fields, logic (conditional visibility, validation, risk scoring), never a published
  version. The agent's output is only ever a starting point a human edits and pushes through
  module 03's own DRAFT → REVIEW → APPROVED → PUBLISHED lifecycle.
- **Default autonomy:** L1 Recommend, structurally capped — this agent has **no tool** that can
  transition a template past `DRAFT`; publishing is always a human, non-agent action (this is a
  hard product boundary, not merely a policy default, because a template controls what
  thousands of frontline submissions look like).
- **Escalation conditions:** N/A for autonomy (capped at draft creation); if the description implies
  logic the builder's schema cannot express (e.g., an unsupported conditional), the agent must
  surface an explicit "unsupported" note in the draft rather than silently omitting the logic.

## 5. Deterministic-Rule + AI Pattern (FR-AGENT-020)

- **FR-AGENT-020:** The runtime MUST route explicit policy thresholds and mandatory actions (e.g.,
  Appendix B's `TEMP-001` rule: temperature > 8 → set_risk HIGH, require corrective_action,
  create_issue, notify_role) through the deterministic **rule engine**, evaluated before any agent
  is invoked. Rules are not "agent behavior" — they are guaranteed outcomes regardless of model
  availability, cost, or output variance.
- **FR-AGENT-021:** Statistical/ML models (e.g., anomaly detection over time-series compliance
  data) are used for forecasting and pattern detection that cannot be expressed as a fixed
  threshold; their output feeds the agent's context (as a `retrieved_sources`-style signal), it
  does not itself authorize an action.
- **FR-AGENT-022:** LLMs are used only for contextual interpretation, summarization,
  retrieval-assisted reasoning, and tool selection **within the boundaries already set by the rule
  engine and the OPA policy**. An LLM output can propose a tool call; it cannot itself decide the
  tool call is authorized — that decision belongs to the deterministic Action Authorization stage
  (§3, stage 8). This is the platform's non-negotiable governance invariant: **no single LLM
  output silently overrides an enterprise policy.**
- **FR-AGENT-023:** When a deterministic rule and an agent's recommendation conflict (e.g., the
  rule engine already set `risk=HIGH` and mandated `create_issue`, and the Compliance Monitor
  independently recommends a different severity), the deterministic rule's outcome is authoritative
  for the mandated fields; the agent's output is treated as a *complementary* recommendation
  (e.g., root cause, additional action) layered on top, never a replacement.

## 6. Human-in-the-Loop: The Agent Inbox

### 6.1 Purpose
The Agent Inbox is the single, consistent surface across all 9 agents where a human reviews and
decides on any action that policy has routed to L3 (or flagged for review at any level due to low
confidence). It must never require the approver to understand the underlying agent's internals to
make a safe decision.

### 6.2 Approval Card — Required Contents (FR-AGENT-201)
Every pending item in the Agent Inbox MUST render an approval card with, at minimum:

| Field | Description |
|---|---|
| Agent identity | Which of the 9 agents proposed this, with `agent_version`. |
| Evidence | The concrete evidence backing the recommendation — submission excerpt, photo/violation bounding box, prior failure history, KnowledgeDocument citation(s) — rendered inline, not just linked. |
| Policy basis | The specific rule/policy clause that makes this an approval-required action (e.g., "severity=CRITICAL" from Appendix C `approval_required_for`), or "flagged: confidence 0.62 < threshold 0.75" for low-confidence flags. |
| Recommendation | The agent's proposed action in plain operational language ("Create a corrective task for Store #204: recalibrate walk-in cooler thermostat, due in 24h"), not raw tool-call JSON as the primary view (raw payload available in an expandable "technical detail" section). |
| Confidence | Numeric confidence score with a visual indicator (e.g., high/medium/low band), and — where applicable — a one-line explanation of what drives the score. |
| Impacted objects | Explicit list of objects the action would touch (WorkItem, Issue, User, Location) with deep links, so the approver can independently verify context before deciding. |
| Proposed action(s) | The exact tool call(s) that would execute on approval, in both plain-language and structured form. |
| Actions available | Approve, Reject, Modify, Request More Evidence (§6.3). |

### 6.3 Approver Actions (FR-AGENT-202)
- **Approve:** Executes the exact proposed tool call(s) as authorized; recorded as
  `human_approval=APPROVED` on the `AgentRun`; proceeds to Tool Execution (loop stage 9).
- **Reject:** No tool call executes; recorded as `human_approval=REJECTED` with an optional reason
  code/free-text; run closes as `REJECTED`; feeds the AI Override Rate metric (source-spec §14.2).
- **Modify:** Approver edits specific fields of the proposed action (e.g., changes the assignee or
  due date on a proposed `create_task`) before execution; the *modified* arguments are what
  execute and what is recorded — the original AI-proposed arguments are retained alongside the
  modified ones in the audit record for evaluation (feeds "human override rate" per source-spec
  §20.5). Modification is constrained to the same tool's argument schema — an approver cannot use
  "Modify" to authorize a different, unproposed tool.
- **Request More Evidence:** Sends the run back into the loop with an explicit instruction to
  re-run Context Builder/Retrieval with an expanded scope (e.g., longer history window) or to
  prompt the agent for an explanation of a specific claim; the original card is marked `PENDING_MORE_EVIDENCE`
  and a new card supersedes it once the agent responds; this does not execute any tool call.

### 6.4 Queue Behavior and Prioritization (FR-AGENT-203)
- The Inbox queue is scoped by the viewer's UserScope (a Store Manager sees only items impacting
  their location(s); a Regional Manager sees their territory roll-up).
- Default sort: severity (from policy basis) descending, then age ascending (oldest pending first)
  to respect the approval-SLA (NFR, §8).
- Items are grouped by triggering agent by default, with a flat "all pending" view available.
- A low-confidence-flagged item (routed for review even at an autonomy level that would otherwise
  auto-execute) is visually distinguished from a policy-mandated L3 approval, since the approver's
  question differs ("is this right?" vs. "do you authorize this?").

## 7. Kill Switch and Agent Policy Configuration UX (Admin)

### 7.1 Kill Switch (FR-AGENT-301)
- **Scope levels:** Tenant-wide (all agents disabled), agent-level (one of the 9 disabled
  tenant-wide), and tool-level (one tool disabled across all agents that could call it, e.g.
  disable `send_notification` platform-wide during an incident) — matches source-spec §20.1
  "Kill switch: Tenant/agent/tool-level disable control."
- **Effect:** A kill-switched scope stops at loop stage 3 (Policy Check) for any new run — no
  model call, no cost, no tool execution. In-flight runs at the moment of kill-switch engagement
  are handled per §9.4 (policy-changed-mid-flight edge case): any not-yet-executed tool call is
  blocked; any run already past Tool Execution for that specific call completes/logs normally.
- **UX:** A prominent, single-click "Engage Kill Switch" control in the Admin Console's Agent
  Governance area, requiring a confirmation step and a mandatory reason (free text, stored on the
  `AgentKillSwitch` record for audit). While engaged, a persistent **kill-switch-engaged banner**
  (§8.3) is shown to every admin/manager persona whose scope is affected, and the affected agent(s)
  show a disabled/greyed state in the Agent Inbox filters.
- **Who can engage:** Enterprise Admin (tenant/agent/tool scope) is the only role with access to the
  true, fail-closed kill switch described here. Operations Admin instead gets a narrower "pause agent
  X for org-node Y" capability, implemented as an `autonomy: L0` policy override scoped to that
  org-node (a normal policy write, not a second kill-switch primitive) — see §12 Resolved Decisions
  for the rationale (keeping exactly one fail-closed mechanism in the system).
- **Re-enable:** Requires the same role, and is itself an audited action (who, when, why).

### 7.2 Agent Policy Configuration (FR-AGENT-302)
Admins configure, per agent per tenant (optionally per org-node scope for large enterprises), a
policy document in the shape of Appendix C:

```
agent: <agent_id>
autonomy: <L0|L1|L2|L3|L4 ceiling for this tenant/scope>
allowed_tools: [...]
blocked_tools: [...]
approval_required_for:
  - <condition expression>
```

- **UX requirements:**
  - The tool picker for `allowed_tools`/`blocked_tools` is constrained to the agent's own catalog
    entries (§ tool catalog in low-level-design.md) — an admin cannot allowlist a tool the agent
    was never built to call.
  - `blocked_tools` always wins over `allowed_tools` if both somehow list the same tool (explicit
    precedence rule, validated at save time with an inline warning, not a silent resolution).
  - `approval_required_for` conditions are built from a structured condition builder (field,
    operator, value) over a fixed set of exposed fields (severity, confidence, cost estimate,
    audience size, external-facing flag, org-node scope) — not free-form Rego text in the primary
    UI (a "view compiled policy" advanced panel may show the generated Rego for transparency/
    export, read-only).
  - The autonomy ceiling set here can only be **equal to or lower than** the platform-wide default
    ceiling for that agent (an Enterprise Admin can restrict further, never exceed the platform's
    hard L4-prohibited boundaries — e.g., no tenant can ever configure `delete_submission` or
    `modify_policy` as an allowed, unapproved tool; those remain L4 platform-wide).
  - **Impact preview:** Before saving a policy change, the admin sees which pending Agent Inbox
    items would be affected (e.g., items that would now auto-execute instead of requiring
    approval) — mirrors the Impact Preview pattern from source-spec §8.4 for workflow publishing.
  - Every policy save creates a new immutable `AgentPolicy` version; agent runs record which
    policy version authorized them (`policy_decision` in the Agent Run Record).

## 8. UX States

### 8.1 Pending-Approval Queue
- **Empty state:** "No agent actions waiting on you" with a link to the Agent Evaluation summary
  (reinforces that agents are working even when nothing needs approval).
- **Loading state:** Skeleton cards; queue count badge shows last-known count until refreshed.
- **Populated state:** Cards per §6.2/6.4, with unread/new indicators for items added since last
  view.
- **Offline/degraded:** If the Inbox cannot reach the server (mobile offline, or backend
  degradation), show cached last-known queue with a clear "may be out of date" banner; approve/
  reject actions taken offline are queued and reconciled on reconnect using the same
  idempotency-key discipline as other mutations — but a **Modify** action requiring fresh field
  validation against current object state is disabled offline (cannot safely validate against
  possibly-changed server state).
- **Permission-denied:** A user without Inbox scope for a given item never sees it in their query
  results (scoped server-side, not hidden client-side).

### 8.2 Agent-Action-Executed Audit Trail View
- Chronological, filterable (by agent, tenant scope/org-node, date range, outcome) list of
  completed `AgentRun`s, each expandable to the full loop trace: context hash, retrieved sources,
  reasoning summary, tools called with arguments/results, policy decision, human approval record
  (if any), execution result, latency, cost.
- **Empty state:** "No agent activity in this range" (distinguish from "agents disabled" which
  shows the kill-switch banner instead).
- Every row deep-links to the impacted object (WorkItem, Issue, etc.) in its owning module.

### 8.3 Kill-Switch-Engaged Banner
- Persistent, non-dismissible-until-resolved banner at the top of Admin Console and Manager
  Control Tower surfaces within the affected scope: "Agent actions are paused for [scope] since
  [time] — reason: [reason]. [Manage]" linking to §7.1.
- Mobile equivalent: a compact banner on the manager's home screen only (not surfaced to
  Frontline Associates, who have no agent-approval role).

### 8.4 Low-Confidence Flagged for Review
- Visually distinct card treatment in the Inbox (e.g., an "AI wants a second opinion" framing
  distinct from "AI needs your authorization") — sets the correct approver mental model per §6.4.
- Includes an explicit confidence-threshold comparison ("scored 0.62, review threshold 0.75") so
  the approver understands *why* it's here even though the underlying tool would otherwise have
  auto-executed at the agent's configured autonomy level.

## 9. Validation and Edge Cases

- **FR-AGENT-401 — Stale context after concurrent human edit:** If the object(s) an agent's
  recommendation targets were modified (by a human or another agent) after the run's context was
  built but before a pending approval is acted on, the system MUST re-validate the target object's
  current version/ETag at approval time. If it has changed, the approval card is marked
  `STALE — CONTEXT CHANGED`, the Approve action is disabled, and the approver is offered
  "Re-evaluate" (re-runs the agent with fresh context) instead of blindly executing a now-outdated
  action. This mirrors the platform's general optimistic-concurrency discipline
  (conventions.md — ETag/version checks).
- **FR-AGENT-402 — Two agents proposing conflicting actions on the same object:** The runtime
  acquires a short-lived advisory lock on the target object (see low-level-design.md concurrency
  section) for the duration of Tool Selection→Tool Execution. A second agent run that reaches Tool
  Selection for the same object while the lock is held is queued (not failed) and re-evaluates
  against the post-first-action state once the lock releases — this may make its original
  recommendation moot (e.g., Issue already resolved), in which case it self-cancels with an
  audited `SUPERSEDED` outcome rather than presenting a stale card to a human.
- **FR-AGENT-403 — Tool execution partial failure requiring compensating action:** If a tool call
  that is one of several in a proposed action set succeeds and a subsequent one fails (e.g.,
  `create_issue` succeeds, `send_notification` fails), the run does NOT silently report overall
  success. Result Validation (loop stage 10) records the partial state; the runtime attempts a
  configured compensating action for the specific tool (e.g., retry `send_notification` with
  backoff per platform reliability patterns) and, if still failing after retry budget, surfaces a
  `PARTIAL_FAILURE` outcome in the audit trail and, if the unresolved failure has material impact
  (e.g., a required notification never went out), escalates to the Agent Inbox as
  "needs attention" rather than closing the run silently as complete.
- **FR-AGENT-404 — Policy changed mid-flight of an in-progress run:** A policy version is
  captured at loop stage 3 (Policy Check) and is the policy that governs the *entire* run,
  including a later Tool Execution after human approval — a policy change that occurs after a
  run's Policy Check but before its Tool Execution does NOT retroactively re-authorize or
  de-authorize an already-approved action; however, if the change was a kill-switch engagement
  (§7.1), that always takes precedence and blocks the not-yet-executed tool call regardless of
  the run's captured policy version (fail-closed for kill switches specifically, since they exist
  precisely for emergency stop scenarios).
- **FR-AGENT-405 — Model/tool-selection produces an unparseable or off-contract output:** Treated
  as a run failure at stage 6 (Reasoning), logged, and — depending on agent criticality — either
  retried once with a stricter structured-output constraint or surfaced as a
  `RUN_FAILED — MODEL_OUTPUT_INVALID` entry in the audit trail; never silently ignored, and never
  passed through to Tool Selection with best-effort guessing.
- **FR-AGENT-406 — Tool argument references an object outside the agent's authorized scope:**
  Even if the model proposes a syntactically valid tool call, Action Authorization (stage 8) MUST
  re-verify that every object ID in the arguments is within the run's authorized tenant/org-node
  scope (defense in depth against prompt injection or retrieval contamination surfacing an
  out-of-scope object ID) — a violation blocks the call, logs a policy violation, and does not
  merely "let the downstream API's own authorization catch it," since the downstream API call
  will be made with a service identity, not the original user's identity.

## 10. Non-Functional Requirements (module-specific)

| NFR | Target |
|---|---|
| Agent decision latency budget | p95 end-to-end loop (event → decision recorded) ≤ 8s for L0/L1/L2 synchronous-path agents (Compliance Monitor, Resolution Agent, Visual Inspection Agent reacting to a single event); Operations Copilot interactive query p95 ≤ 4s (perceived responsiveness matters more for a conversational surface); Executive Insights/Audit Intelligence batch/scheduled runs are exempt from the interactive budget but must complete within their scheduling window. |
| Human-approval SLA | Default: CRITICAL-severity approvals must be surfaced to at least one authorized approver's notification channel within 60s of the run parking in the Inbox; if unactioned within a tenant-configured window (default 2h for CRITICAL, 24h otherwise), the item escalates up the org hierarchy per the same escalation-ladder mechanism used for Issue SLAs (source-spec §7.2/§10.2), and a background job (low-level-design.md) drives this. |
| Audit completeness | 100% of tool executions MUST be logged to `AgentToolExecution` before the run is marked complete — this is enforced structurally (the execution call path cannot commit without a preceding audit write in the same transaction/outbox) not just as a best-effort logging call; a failed audit write blocks the tool execution from being considered final and forces a `RUN_FAILED — AUDIT_WRITE_FAILED` state rather than an unaudited success. |
| Policy-check availability | Because Policy Check is on the critical path and fails closed (high-level-design.md), its p99 latency budget is ≤150ms from a warm cache; a cold/degraded policy engine may add up to a configured circuit-breaker timeout before the run fails closed. |
| Confidence-threshold consistency | Confidence thresholds that route to human review must be configurable per tenant per agent but have platform-wide sane defaults (0.75 general, 0.9 for auto-execute-eligible CRITICAL-adjacent categories) so a misconfigured tenant cannot accidentally set a 0.0 threshold and defeat the review mechanism for the CRITICAL/L3-mandatory categories — those remain policy-forced regardless of confidence. |

## 11. Dependencies

The Agent Runtime is a **typed tool-calling client of nearly every other module**. This is
intentional and central to the architecture — it must never gain a private write path into
another module's data.

| Dependency | Direction | Nature |
|---|---|---|
| Work Orchestration (module 02) | Agent → module API | Tools: `create_task`, `assign_task`, `get_task_status`. Consumes `task.created`, `task.overdue` events as triggers. |
| Issues/CAPA (module 06) | Agent → module API | Tools: `create_issue`, and (CAPA-specific extensions in low-level-design.md tool catalog) linking corrective actions. Consumes `issue.created`, `issue.escalated`, `issue.resolved` as triggers. |
| Communications/Knowledge (module 07) | Agent → module API | Tools: `send_notification`, `search_knowledge`; Communication Agent drafts feed module 07's publication pipeline (draft state owned by this module until "send," publication mechanics owned by module 07). Consumes `message.published`, `message.acknowledged`. |
| Learning (module 08) | Agent → module API | Tools: `schedule_training`; reads Course/LearningAssignment catalog. Consumes `training.completed` to measure outcome improvement (source-spec §11.3). |
| Visual AI (module 09) | Event consumer + Agent → module API (read) | Consumes visual-AI result-available events; reads the result contract (`compliance_score, confidence, status, violations[]`); does not write back into module 09's own review state. |
| Evidence (module 05) | Agent → module API (read) | Tools: `get_submission` and evidence lookups feeding context/evidence-refs on approval cards; read-only, signed-URL access respecting the same retention/encryption controls as human access. |
| Identity/Org/Authorization (module 01) | Agent ← platform service (read) | Context Builder resolves tenant/org-node/user scope via the Authorization Service; the Agent Runtime never re-implements scope logic. |
| Analytics/Control Tower | Agent → event/metric producer | Emits `agent.invoked`, `agent.tool_executed`, `agent.action_approved`, `agent.action_rejected` (source-spec §18.3) and the Agent Evaluation metric set (§20.5) consumed by dashboards owned elsewhere. |
| Event Backbone | Both directions | Nearly every canonical event in source-spec §18.3 is a potential trigger; every agent action emits its own canonical events. |

## 12. Resolved Decisions

Formerly "Open Questions/Assumptions." Each item below was an open question in the prior revision;
each now carries a firm, build-ready decision (see `_context/engineering-decisions.md` for the
platform-wide decisions this module inherits rather than restates).

- **Deployment shape:** All 9 agents share one Agent Runtime service/deployment, not independently
  deployed microservices per agent — they share the identical loop pipeline, policy engine, and audit
  contract; only prompts/tool allowlists differ. Independent horizontal scaling is achieved via
  per-agent-type worker pools within the shared service (high-level-design.md §5), not separate
  services. This is final for the initial build; re-splitting a specific agent (e.g., Visual Inspection
  Agent under sustained high image volume) into its own deployment is an operational scaling decision
  made later from real load data, not a spec ambiguity blocking build start.
- **Kill-switch scoping:** Enterprise Admin owns tenant/agent/tool-level kill switches (the true,
  fail-closed emergency stop). Operations Admin gets a narrower "pause agent X for org-node Y"
  capability, implemented as an `autonomy: L0` policy override scoped to that org-node — a normal
  policy write, not a second kill-switch primitive. This keeps exactly one fail-closed mechanism in the
  system (simpler to reason about and to test under chaos conditions, engineering-decisions.md §7).
- **Operations Copilot memory:** No persistent cross-session memory by default (matches the Agent
  Contract memory policy of explicit retention/scoping). A tenant may opt in to session-continuity
  memory scoped to one user with an explicit retention window (default 30 days if enabled); this is
  never shared across users and never used as retrieval context for a different user's query under any
  configuration.
- **Confidence-threshold defaults:** 0.75 general / 0.9 for auto-execute-eligible CRITICAL-adjacent
  categories (§10 NFR table) are the shipped v1 defaults. They are tenant-configurable upward
  (stricter) and downward within a platform-enforced floor of 0.5 (a tenant cannot configure a
  threshold that effectively disables review); recalibration against real production data is an
  ongoing model-ops activity (owned by the Agent Governance surface, §8), not a blocker to shipping v1
  with these defaults.
- **Workflow Builder Agent's publish boundary:** Permanent product boundary, not a future-autonomy-level
  target. This agent can never call the publish endpoint at any autonomy level; a future increase in
  its autonomy (source-spec §25.4 "Autonomous Ops") can only ever mean faster/better drafts and
  auto-routing to the right reviewer, never draft-to-publish without a human — enforced at the tool
  catalog level (the `publish` tool is simply never in this agent's allowlist, at any policy
  configuration, tenant override included).

## 13. EU AI Act Alignment — Employment/Worker-Management AI (Binding)

Per `_context/engineering-decisions.md` §12: AI systems used for employment/worker-management
(task allocation, performance monitoring, promotion-relevant evaluation) are classified **high-risk**
under the EU AI Act (Annex III §4), with obligations enforceable from August 2, 2026. This platform
implements the following controls globally (all tenants, not EU-conditional — rationale in the
register) as hard requirements on this module:

### FR-AGENT-500 — Mandatory Human Approval for Individual-Employee-Affecting Actions
Any agent action that would materially affect a **named individual employee's** performance record,
standing, disciplinary status, or continued employment (as distinct from a location- or
process-level recommendation with no single named employee as its subject) **always** requires human
approval before taking effect, regardless of that agent's configured autonomy level — this floor is
enforced at Action Authorization (loop stage 8, §3) as a policy condition that cannot be
lowered by any tenant configuration; an Enterprise Admin may add further categories requiring
approval, never remove this one. Classification of "individual-employee-affecting" is itself a
deterministic tag on the target tool/object (e.g., `create_performance_flag`,
`update_employee_standing`), not a judgment call left to the model.

### FR-AGENT-501 — Extended Audit Retention for Individual-Employee-Affecting Decisions
`AgentRun`/`AgentToolExecution` records for any run that reached an FR-AGENT-500 approval gate are
retained a minimum of **3 years** (longer than the platform's general audit-log retention, per module
13's Security/Privacy schedule), consistent with common labor-record retention norms; the retained
record is the decision record (context hash, policy version, human decision, timestamp), not the full
retrieval/context payload, once past 1 year — balancing EU AI Act Article 12 traceability against
GDPR data-minimization.

### FR-AGENT-502 — "AI in This Workplace" Transparency Notice
The mobile app's account/settings surface exposes an "AI in this Workplace" notice, per tenant,
listing every currently-active agent for that tenant, a plain-language description of what it does
(sourced from that agent's own persona description, §2.2), and a contact/appeal path for an employee
to request human review of a specific agent-driven decision that affected them. This is a hard
requirement (built and shown by default), not an admin-configurable toggle, satisfying EU AI Act
Article 13 transparency without requiring per-tenant jurisdiction detection logic.

### FR-AGENT-503 — Per-Agent Risk Assessment Artifact
The Agent Governance surface (§8, Enterprise Admin) requires a structured risk-assessment record
before an agent can be enabled for a tenant: `{agent_id, intended_purpose, affected_persona_categories[],
known_failure_modes[], mitigations[], human_oversight_mechanism, last_reviewed_at, reviewed_by}`. This
record is re-required (a stale one blocks further action) at minimum annually or immediately upon any
autonomy-level increase for that agent. This operationalizes EU AI Act Article 9's risk-management
process as a concrete, testable data object rather than an off-platform compliance document that can
drift from what the system actually does.
