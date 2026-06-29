# PLAN — akka-hitl-gate

Architectural sketch for HITL Ops Automation Gate. All four mermaid diagrams plus the component table.

---

## Component graph

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#0e1e2a", "primaryBorderColor": "#7EC8E3", "primaryTextColor": "#7EC8E3", "lineColor": "#888888", "edgeLabelBackground": "#111111"}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  EP[GateEndpoint]:::ep
  APP[AppEndpoint]:::ep
  WF[GateWorkflow]:::wf
  PA[ProposalAgent]:::agent
  EA[ExecutionAgent]:::agent
  AE[ActionEntity]:::ese
  AV[ActionsView]:::view

  EP -->|action-request| WF
  WF -->|propose task| PA
  WF -->|execute task| EA
  PA -->|recordProposal| AE
  EA -->|recordExecution| AE
  EP -->|approve / reject| AE
  WF -->|poll status| AE
  AE -.->|events| AV
  EP -->|getAllActions / SSE| AV
  APP -->|static UI| EP
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor Operator
  participant EP as GateEndpoint
  participant WF as GateWorkflow
  participant PA as ProposalAgent
  participant AE as ActionEntity
  participant EA as ExecutionAgent

  Operator->>EP: POST /api/action-request {taskDescription}
  EP->>WF: start(actionId, taskDescription)
  WF->>PA: runSingleTask(PROPOSE)
  PA-->>WF: ActionProposal{summary, estimatedImpact, actionPayload}
  WF->>AE: recordProposal -> PROPOSED
  Note over WF,AE: await-approval task paused; workflow polls status every 5s
  Operator->>EP: POST /api/actions/{id}/approve
  EP->>AE: approve -> APPROVED
  WF->>AE: getAction -> APPROVED
  WF->>EA: runSingleTask(EXECUTE) [guard: status == APPROVED]
  EA-->>WF: ActionResult{outcome, executedAt}
  WF->>AE: recordExecution -> EXECUTED
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> PROPOSED: ActionProposed
  PROPOSED --> APPROVED: ActionApproved
  PROPOSED --> REJECTED: ActionRejected
  APPROVED --> EXECUTED: ActionExecuted
  REJECTED --> [*]
  EXECUTED --> [*]
```

## Entity model

```mermaid
erDiagram
  ACTION ||--o{ ACTION_EVENT : emits
  ACTION {
    string id
    string taskDescription
    string status
    string proposalSummary
    string estimatedImpact
    string actionPayload
    string outcome
  }
  ACTION_EVENT {
    string type
    string occurredAt
  }
  ACTIONS_VIEW {
    string id
    string status
  }
  ACTION ||--|| ACTIONS_VIEW : projects
```

## Component table

| Component | Path (generated) |
|---|---|
| ProposalAgent | `application/ProposalAgent.java` |
| ExecutionAgent | `application/ExecutionAgent.java` |
| GateWorkflow | `application/GateWorkflow.java` |
| GateTasks | `application/GateTasks.java` |
| ActionEntity | `application/ActionEntity.java` |
| ActionsView | `application/ActionsView.java` |
| GateEndpoint | `api/GateEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| Action / events / records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `proposeStep` and `executeStep` call agents; both set `stepTimeout(60s)` to absorb LLM latency. The default 5 s step timeout would retry forever (Lesson 4).
- **Await-approval task.** The workflow does not block a thread; `awaitApprovalStep` reads `ActionEntity.getAction`, and on `PROPOSED` self-schedules a 5-second resume timer until the human transitions the status.
- **Idempotency.** `actionId` is the workflow id and the entity id; re-delivery of `recordProposal` / `recordExecution` is absorbed by event-applier checks on current status.
- **Execution guard.** Before the execution tool runs, the before-tool-call guardrail re-reads `ActionEntity.status`; if it is not `APPROVED`, the call is blocked. No compensation path is needed because execution is the terminal write.
