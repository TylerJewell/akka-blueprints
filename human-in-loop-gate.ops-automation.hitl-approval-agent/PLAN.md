# PLAN — hitl-approval-agent

Architectural sketch for the Human-in-the-Loop Approval Agent. All four mermaid diagrams plus the component table.

---

## Component graph

```mermaid
%%{init: {"theme": "base", "themeVariables": {
  "primaryColor": "#0e1e2a",
  "primaryTextColor": "#7EC8E3",
  "primaryBorderColor": "#7EC8E3",
  "lineColor": "#888888",
  "secondaryColor": "#1f1900",
  "tertiaryColor": "#161616"
}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  EP[ApprovalEndpoint]:::ep
  APP[AppEndpoint]:::ep
  WF[ApprovalWorkflow]:::wf
  PA[ProposalAgent]:::agent
  EA[ExecutionAgent]:::agent
  AE[ActionEntity]:::ese
  AV[ActionsView]:::view

  EP -->|action-request| WF
  WF -->|propose task| PA
  WF -->|execute task| EA
  PA -->|recordProposal| AE
  EA -->|recordExecution| AE
  EP -->|approve / deny| AE
  WF -->|poll status| AE
  AE -.->|events| AV
  EP -->|getAllActions / SSE| AV
  APP -->|static UI| EP
```

## Interaction sequence

```mermaid
%%{init: {"theme": "base", "themeVariables": {
  "primaryColor": "#0e1e2a",
  "primaryTextColor": "#e2e8f0",
  "primaryBorderColor": "#F5C518",
  "lineColor": "#888888",
  "actorBkg": "#0e1e2a",
  "actorTextColor": "#e2e8f0",
  "actorBorderColor": "#F5C518",
  "noteBkgColor": "#1c1330",
  "noteTextColor": "#A855F7"
}}}%%
sequenceDiagram
  autonumber
  actor Operator
  participant EP as ApprovalEndpoint
  participant WF as ApprovalWorkflow
  participant PA as ProposalAgent
  participant AE as ActionEntity
  participant EA as ExecutionAgent

  Operator->>EP: POST /api/action-request {operationRequest}
  EP->>WF: start(actionId, operationRequest)
  WF->>PA: runSingleTask(PROPOSE)
  PA-->>WF: ActionProposal{actionType, target, rationale, estimatedImpact}
  WF->>AE: recordProposal -> PROPOSED
  Note over WF,AE: await-approval task paused; workflow polls status every 5s
  Operator->>EP: POST /api/actions/{id}/approve
  EP->>AE: approve -> APPROVED
  WF->>AE: getAction -> APPROVED
  WF->>EA: runSingleTask(EXECUTE) [guard: status == APPROVED]
  EA-->>WF: ActionResult{outcome, completedAt, details}
  WF->>AE: recordExecution -> EXECUTED
```

## State machine

```mermaid
%%{init: {"theme": "base", "themeVariables": {
  "primaryColor": "#1f1900",
  "primaryTextColor": "#F5C518",
  "primaryBorderColor": "#F5C518",
  "lineColor": "#888888",
  "secondaryColor": "#0e2010",
  "tertiaryColor": "#161616",
  "transitionLabelColor": "#cccccc"
}}}%%
stateDiagram-v2
  [*] --> PROPOSED: ActionProposed
  PROPOSED --> APPROVED: ActionApproved
  PROPOSED --> DENIED: ActionDenied
  APPROVED --> EXECUTED: ActionExecuted
  DENIED --> [*]
  EXECUTED --> [*]
```

<style>
  .statediagram-state rect { fill: #1f1900; stroke: #F5C518; }
  .statediagram-state text { fill: #F5C518 !important; }
  .edgeLabel foreignObject { overflow: visible; }
  .transition text { fill: #cccccc; }
</style>

## Entity model

```mermaid
%%{init: {"theme": "base", "themeVariables": {
  "primaryColor": "#1f1900",
  "primaryTextColor": "#F5C518",
  "primaryBorderColor": "#F5C518",
  "lineColor": "#888888"
}}}%%
erDiagram
  ACTION ||--o{ ACTION_EVENT : emits
  ACTION {
    string id
    string operationRequest
    string status
    string actionType
    string target
    string rationale
    string estimatedImpact
    string outcome
  }
  ACTION_EVENT {
    string type
    string occurredAt
  }
  ACTIONS_VIEW {
    string id
    string status
    string actionType
    string target
  }
  ACTION ||--|| ACTIONS_VIEW : projects
```

## Component table

| Component | Path (generated) |
|---|---|
| ProposalAgent | `application/ProposalAgent.java` |
| ExecutionAgent | `application/ExecutionAgent.java` |
| ApprovalWorkflow | `application/ApprovalWorkflow.java` |
| ApprovalTasks | `application/ApprovalTasks.java` |
| ActionEntity | `application/ActionEntity.java` |
| ActionsView | `application/ActionsView.java` |
| ApprovalEndpoint | `api/ApprovalEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| Action / events / records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `proposeStep` and `executeStep` call agents; both set `stepTimeout(60s)` to absorb LLM latency. The default 5 s step timeout would expire before most models respond (Lesson 4).
- **Await-approval task.** The workflow does not block a thread; `awaitApprovalStep` reads `ActionEntity.getAction`, and on `PROPOSED` self-schedules a 5-second resume timer until the operator transitions the status.
- **Idempotency.** `actionId` is the workflow id and the entity id; re-delivery of `recordProposal` / `recordExecution` is absorbed by event-applier checks on current status.
- **Execution guard.** Before the execution tool runs, the before-tool-call guardrail re-reads `ActionEntity.status`; if it is not `APPROVED`, the call is blocked. No compensation path is needed because execution is the terminal write.
