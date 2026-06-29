# PLAN — tool-call-approval

Architectural sketch for HITL Tool Approval. All four mermaid diagrams plus the component table.

---

## Component graph

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#0e1e2a', 'edgeLabelBackground': '#161616', 'transitionLabelColor': '#cccccc'}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  EP[ApprovalEndpoint]:::ep
  APP[AppEndpoint]:::ep
  WF[ApprovalWorkflow]:::wf
  PA[PlannerAgent]:::agent
  EA[ExecutorAgent]:::agent
  TE[ToolRequestEntity]:::ese
  TV[ToolRequestsView]:::view

  EP -->|tool-requests| WF
  WF -->|plan task| PA
  WF -->|execute task| EA
  PA -->|recordPlan| TE
  EA -->|recordExecution| TE
  EP -->|approve / reject| TE
  WF -->|poll status| TE
  TE -.->|events| TV
  EP -->|getAllRequests / SSE| TV
  APP -->|static UI| EP
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor Operator
  participant EP as ApprovalEndpoint
  participant WF as ApprovalWorkflow
  participant PA as PlannerAgent
  participant TE as ToolRequestEntity
  participant EA as ExecutorAgent

  Operator->>EP: POST /api/tool-requests {goal}
  EP->>WF: start(requestId, goal)
  WF->>PA: runSingleTask(PLAN)
  PA-->>WF: ToolCallPlan{toolName, parameters, rationale}
  WF->>TE: recordPlan -> PENDING_APPROVAL
  Note over WF,TE: await-approval task paused; workflow polls status every 5s
  Operator->>EP: POST /api/tool-requests/{id}/approve {editedParameters?}
  EP->>TE: approve -> APPROVED
  WF->>TE: getRequest -> APPROVED
  WF->>EA: runSingleTask(EXECUTE) [guard: status == APPROVED]
  EA-->>WF: ToolCallResult{output, executedAt}
  WF->>TE: recordExecution -> EXECUTED
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> PENDING_APPROVAL: ToolCallPlanned
  PENDING_APPROVAL --> APPROVED: ToolCallApproved
  PENDING_APPROVAL --> REJECTED: ToolCallRejected
  APPROVED --> EXECUTED: ToolCallExecuted
  REJECTED --> [*]
  EXECUTED --> [*]
```

## Entity model

```mermaid
erDiagram
  TOOL_REQUEST ||--o{ TOOL_REQUEST_EVENT : emits
  TOOL_REQUEST {
    string id
    string goal
    string status
    string toolName
    string parameters
    string executionOutput
  }
  TOOL_REQUEST_EVENT {
    string type
    string occurredAt
  }
  TOOL_REQUESTS_VIEW {
    string id
    string status
  }
  TOOL_REQUEST ||--|| TOOL_REQUESTS_VIEW : projects
```

## Component table

| Component | Path (generated) |
|---|---|
| PlannerAgent | `application/PlannerAgent.java` |
| ExecutorAgent | `application/ExecutorAgent.java` |
| ApprovalWorkflow | `application/ApprovalWorkflow.java` |
| ApprovalTasks | `application/ApprovalTasks.java` |
| ToolRequestEntity | `application/ToolRequestEntity.java` |
| ToolRequestsView | `application/ToolRequestsView.java` |
| ApprovalEndpoint | `api/ApprovalEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| ToolRequest / events / records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `planStep` and `executeStep` call agents; both set `stepTimeout(60s)` to absorb LLM latency. The default 5 s step timeout would cause premature retries (Lesson 4).
- **Await-approval task.** The workflow does not block a thread; `awaitApprovalStep` reads `ToolRequestEntity.getRequest`, and on `PENDING_APPROVAL` self-schedules a 5-second resume timer until the human transitions the status.
- **Parameter editing.** When the operator approves with `editedParameters` present, `ToolCallApproved` records the override in the entity; `executeStep` reads the entity to obtain the effective parameters before calling `ExecutorAgent`.
- **Idempotency.** `requestId` is the workflow id and the entity id; re-delivery of `recordPlan` / `recordExecution` is absorbed by event-applier checks on current status.
- **Execute guard.** Before the execute tool runs, the before-tool-call guardrail re-reads `ToolRequestEntity.status`; if it is not `APPROVED`, the call is blocked. No compensation path is needed because execution is the terminal write.
