# Agentic Operations (Agent Runtime) — Architecture Diagrams

Module: `10-agentic-operations` | Epic: **EPIC-11 Agent Runtime**

Focused diagram set for this module: component/container view, two sequence diagrams (L2
auto-execute and L3 approval flows), an ER diagram of owned entities, and the Agent Autonomy Level
decision flow. Prose is kept to short narration per diagram — full rationale lives in
high-level-design.md and low-level-design.md.

## 1. Component / Container Diagram

```mermaid
flowchart TD
    subgraph Triggers
        EVT[Event Backbone]
        SCHED[Scheduler]
        MANUAL[Manual Invocation]
    end

    subgraph AgentRuntime["Agent Runtime Service"]
        direction TB
        DISPATCH[Run Dispatcher]
        CTX[Context Builder]
        POL[Policy Engine - OPA/Rego]
        GATE[Model Gateway]
        RETR[Retrieval - pgvector/OpenSearch]
        REASON[Reasoning]
        TOOLSEL[Tool Selection]
        AUTHZ[Action Authorization]
        EXEC[Tool Executor]
        VALID[Result Validator]
        AUDIT[Audit Writer]
    end

    INBOX[Agent Inbox]
    TARGETS[Target Module APIs\nWork / Issues / Comms / Learning / Visual AI / Evidence]

    EVT --> DISPATCH
    SCHED --> DISPATCH
    MANUAL --> DISPATCH
    DISPATCH --> CTX --> POL --> GATE --> RETR --> REASON --> TOOLSEL --> AUTHZ
    AUTHZ -->|auto| EXEC
    AUTHZ -->|approval required| INBOX --> EXEC
    AUTHZ -->|blocked| AUDIT
    EXEC --> TARGETS
    EXEC --> VALID --> AUDIT
    AUDIT --> EVT
```

**Narration:** The runtime is one service implementing the fixed pipeline from source-spec §13.2.
It is deliberately drawn as a hub: it consumes almost every domain event as a trigger and calls
almost every other module's write API as a tool, but it owns none of that domain state itself —
only the run/policy/approval/audit records described in §3.

## 2. Sequence Diagram — L2 Auto-Execute Flow

```mermaid
sequenceDiagram
    participant EVT as Event Backbone
    participant RT as Agent Runtime
    participant POL as Policy Engine
    participant TGT as Target Module API
    participant AUDIT as Audit Store

    EVT->>RT: issue.created (repeated failure pattern)
    RT->>POL: policy check + action authorization
    POL-->>RT: AUTO_EXECUTE (L2, no approval_required_for match)
    RT->>TGT: create_issue / create_task (Idempotency-Key)
    TGT-->>RT: 201 Created
    RT->>AUDIT: AgentRun + AgentToolExecution (COMPLETED, human_approval=NOT_REQUIRED)
    RT->>EVT: agent.tool_executed
```

**Narration:** At L2, the deterministic policy check both authorizes and closes the loop in one
pass — no human touches the flow. Every fact (policy version, tool arguments/results) is still
written to the immutable audit trail before the run is marked complete, satisfying the 100%
audit-completeness NFR even when no human was involved.

## 3. Sequence Diagram — L3 Approval Flow

```mermaid
sequenceDiagram
    participant RT as Agent Runtime
    participant POL as Policy Engine
    participant INBOX as Agent Inbox (ApprovalRequest)
    participant MGR as Human Approver
    participant TGT as Target Module API
    participant AUDIT as Audit Store

    RT->>POL: action authorization (e.g. severity=CRITICAL match)
    POL-->>RT: APPROVAL_REQUIRED
    RT->>INBOX: create ApprovalRequest (evidence, policy basis, recommendation, confidence)
    RT->>AUDIT: AgentRun (execution_result=PENDING_APPROVAL)
    MGR->>INBOX: review approval card
    MGR->>INBOX: Approve / Reject / Modify / Request More Evidence
    alt Approved or Modified
        INBOX->>TGT: execute proposed (or modified) tool call
        TGT-->>INBOX: success/failure
        INBOX->>AUDIT: AgentToolExecution + AgentRun updated (COMPLETED)
    else Rejected
        INBOX->>AUDIT: AgentRun updated (REJECTED) - no tool call executes
    else Request More Evidence
        INBOX->>RT: re-run Context Builder / Retrieval with expanded scope
    end
```

**Narration:** L3 is the human-in-the-loop path required by source-spec §13.4. The approval card
carries everything the approver needs (evidence, policy basis, recommendation, confidence,
impacted objects, proposed action) so the decision does not require reading raw agent internals.
A rejection never executes any tool call; a modification executes only the approver's edited
arguments, both outcomes fully audited for the human-override evaluation metric.

## 4. ER Diagram — Owned Entities

```mermaid
erDiagram
    AGENT_POLICIES ||--o{ AGENT_RUNS : governs
    AGENT_RUNS ||--o{ AGENT_TOOL_EXECUTIONS : produces
    AGENT_RUNS ||--o| APPROVAL_REQUESTS : "may create"
    AGENT_KILL_SWITCHES ||--o{ AGENT_RUNS : "may block"

    AGENT_POLICIES {
        uuid id PK
        uuid tenant_id
        text agent_id
        int version
        text autonomy_ceiling
        jsonb allowed_tools
        jsonb blocked_tools
        jsonb approval_required_for
    }
    AGENT_RUNS {
        uuid id PK
        uuid tenant_id
        text agent_id
        text agent_version
        text trigger_type
        numeric confidence
        text policy_decision
        text human_approval
        text execution_result
        uuid trace_id
    }
    AGENT_TOOL_EXECUTIONS {
        uuid id PK
        uuid run_id FK
        text tool_name
        text autonomy_decision
        text outcome
        text idempotency_key
    }
    APPROVAL_REQUESTS {
        uuid id PK
        uuid run_id FK
        text reason_type
        jsonb proposed_tool_calls
        text status
        timestamptz sla_due_at
    }
    AGENT_KILL_SWITCHES {
        uuid id PK
        text scope_type
        text agent_id
        text tool_name
        boolean engaged
    }
```

**Narration:** `AgentPolicy` is the versioned authorization contract every `AgentRun` is stamped
with; `AgentRun` fans out into one or more `AgentToolExecution` rows (the fine-grained audit/
control surface) and, when human review is triggered, exactly one `ApprovalRequest`. Kill switches
sit outside the run lifecycle and can block a run before it ever reaches tool execution.

## 5. Agent Autonomy Level Decision Flow

Reuses the L0–L4 definitions verbatim from conventions.md / source-spec §20.3.

```mermaid
stateDiagram-v2
    [*] --> ToolProposed : Model proposes tool call

    ToolProposed --> KillSwitchCheck
    KillSwitchCheck --> Blocked : tenant/agent/tool kill-switched
    KillSwitchCheck --> PlatformFloorCheck : not killed

    PlatformFloorCheck --> Blocked : tool is L4 Prohibited (platform-wide)
    PlatformFloorCheck --> PolicyAllowlistCheck : not L4

    PolicyAllowlistCheck --> Blocked : tool in blocked_tools OR not in allowed_tools
    PolicyAllowlistCheck --> ScopeCheck : tool allowlisted for this agent/tenant

    ScopeCheck --> Blocked : args reference out-of-scope object
    ScopeCheck --> ConditionCheck : all objects in authorized scope

    ConditionCheck --> ApprovalRequired : matches approval_required_for\n(e.g. severity=CRITICAL)
    ConditionCheck --> ConfidenceCheck : no condition match

    ConfidenceCheck --> ApprovalRequired : confidence < tenant threshold\n(flagged for review)
    ConfidenceCheck --> CeilingCheck : confidence OK

    CeilingCheck --> AutoExecuted : tool's min autonomy <= L2\nand within policy ceiling
    CeilingCheck --> ApprovalRequired : tool's min autonomy == L3
    CeilingCheck --> Blocked : ceiling exceeded, no L3 path configured

    AutoExecuted --> [*] : L2 Execute low-risk
    ApprovalRequired --> HumanDecision : L3 Execute with approval
    HumanDecision --> AutoExecuted : Approved / Modified
    HumanDecision --> Rejected : Rejected
    Rejected --> [*]
    Blocked --> [*] : L4 Prohibited / policy block
```

**Narration:** Every proposed tool call walks this same deterministic gate regardless of which of
the 9 agents proposed it. Kill switches and the platform L4-prohibited floor are checked first and
always win; tenant policy (`blocked_tools`/`allowed_tools`/`approval_required_for`) is evaluated
next; confidence is an independent, additional gate that can force review even for an
otherwise-auto-executable tool. Only after all of these pass does a call reach `AutoExecuted`
(L2) — anything routed to `ApprovalRequired` becomes an Agent Inbox item (L3), and anything
`Blocked` is logged and surfaced, never silently dropped, per the source-spec §24.1 acceptance
test.
