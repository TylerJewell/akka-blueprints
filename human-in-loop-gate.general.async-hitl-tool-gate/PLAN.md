# Implementation Plan — `async-hitl-tool-gate`

The architecture `SPEC.md` resolves to once run through `/akka:specify` → `/akka:plan`. Diagrams render on the Architecture tab. All mermaid blocks use the Akka theme variables plus the Lesson 24 CSS overrides for state-diagram labels (state names white-on-dark, edge labels `overflow:visible`).

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1c1c1c','primaryTextColor':'#ffffff','primaryBorderColor':'#e6c200','lineColor':'#e6c200','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans, sans-serif'}}}%%
flowchart TB
  SIM[RequestSimulator<br/>TimedAction]
  Q[InboundRequestQueue<br/>EventSourcedEntity]
  CON[RequestConsumer<br/>Consumer]
  WF[ApprovalWorkflow<br/>Workflow]
  AA[ActionAgent<br/>AutonomousAgent]
  EA[ExecutorAgent<br/>AutonomousAgent]
  ENT[ActionEntity<br/>EventSourcedEntity]
  VIEW[ActionsView<br/>View]
  MON[StuckActionMonitor<br/>TimedAction]
  EP[ActionEndpoint<br/>HttpEndpoint]
  APP[AppEndpoint<br/>HttpEndpoint]

  SIM -->|enqueueRequest| Q
  Q -.InboundRequestQueued.-> CON
  CON -->|start| WF
  WF -->|runSingleTask PLAN| AA
  AA -->|recordPlan| ENT
  WF -->|runSingleTask EXECUTE| EA
  EA -->|recordExecution| ENT
  ENT -.events.-> VIEW
  MON -->|markEscalated| ENT
  EP -->|approve/reject| ENT
  EP -->|getAllActions / SSE| VIEW
  MON -.getAllActions.-> VIEW
  APP -->|static| EP
```

## Interaction sequence

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1c1c1c','primaryTextColor':'#ffffff','primaryBorderColor':'#e6c200','lineColor':'#e6c200','fontFamily':'Instrument Sans, sans-serif'}}}%%
sequenceDiagram
  participant U as User/Client
  participant EP as ActionEndpoint
  participant WF as ApprovalWorkflow
  participant AA as ActionAgent
  participant ENT as ActionEntity
  participant EA as ExecutorAgent
  U->>EP: POST /api/requests
  EP->>WF: start(actionId, request)
  WF->>AA: runSingleTask(PLAN)
  AA-->>WF: ActionPlan
  WF->>ENT: recordPlan -> PLANNED
  Note over WF,ENT: awaitApprovalStep polls; pauses on PLANNED (5s resume timer)
  U->>EP: POST /api/actions/{id}/approve
  EP->>ENT: approve -> APPROVED
  WF->>ENT: getAction (APPROVED)
  WF->>EA: runSingleTask(EXECUTE)
  Note over EA: before-tool-call guardrail checks status == APPROVED
  EA-->>WF: ToolResult
  WF->>ENT: recordExecution -> EXECUTED
```

## State machine

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1c1c1c','primaryTextColor':'#ffffff','primaryBorderColor':'#e6c200','lineColor':'#e6c200','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc','fontFamily':'Instrument Sans, sans-serif'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> PLANNED: ActionPlanned
  PLANNED --> APPROVED: ActionApproved
  PLANNED --> REJECTED: ActionRejected
  PLANNED --> ESCALATED: ActionEscalated
  APPROVED --> EXECUTED: ActionExecuted
  REJECTED --> [*]
  ESCALATED --> [*]
  EXECUTED --> [*]
```

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1c1c1c','primaryTextColor':'#ffffff','primaryBorderColor':'#e6c200','lineColor':'#e6c200','fontFamily':'Instrument Sans, sans-serif'}}}%%
erDiagram
  INBOUND_REQUEST_QUEUE ||--o{ ACTION : "starts workflow"
  ACTION ||--o{ ACTION_EVENT : "emits"
  ACTION ||--|| ACTIONS_VIEW : "projects to"
  ACTION {
    string id
    string request
    enum status
    instant plannedAt
    string toolName
    string toolArguments
    string rationale
    instant approvedAt
    instant executedAt
    string toolOutput
  }
  ACTION_EVENT {
    string type
    instant timestamp
  }
  ACTIONS_VIEW {
    string id
    enum status
  }
```

## Component table

| Component | Kind | File |
|---|---|---|
| `ActionAgent` | AutonomousAgent | `application/ActionAgent.java` |
| `ExecutorAgent` | AutonomousAgent | `application/ExecutorAgent.java` |
| `ApprovalTasks` | task definitions | `application/ApprovalTasks.java` |
| `ActionEntity` | EventSourcedEntity | `application/ActionEntity.java` |
| `InboundRequestQueue` | EventSourcedEntity | `application/InboundRequestQueue.java` |
| `ActionsView` | View | `application/ActionsView.java` |
| `ApprovalWorkflow` | Workflow | `application/ApprovalWorkflow.java` |
| `RequestConsumer` | Consumer | `application/RequestConsumer.java` |
| `RequestSimulator` | TimedAction | `application/RequestSimulator.java` |
| `StuckActionMonitor` | TimedAction | `application/StuckActionMonitor.java` |
| `ActionEndpoint` | HttpEndpoint | `api/ActionEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |
| `Bootstrap` | service-setup | `Bootstrap.java` |
| `Action`, `ActionStatus`, `ActionEvent` | domain | `domain/*.java` |

Validator component count: **2 http-endpoint · 2 timed-action · 1 view · 1 workflow · 1 service-setup · 2 autonomous-agent · 1 consumer · 2 event-sourced-entity**.

## Concurrency notes

- **Step timeouts (Lesson 4):** `planStep` and `executeStep` 60 s (LLM calls); `awaitApprovalStep` 10 s, which re-arms a 5 s resume timer while status is `PLANNED`. Default 5 s would time out agent steps.
- **Idempotency:** the workflow id is the `actionId` (a fresh UUID per inbound request), so a redelivered `InboundRequestQueued` event maps to a distinct workflow instance; `ActionEntity` command handlers are no-ops when the target status is already reached.
- **No saga/compensation:** the only side-effecting step is the simulated tool call, gated behind the `APPROVED` status and the before-tool-call guardrail; rejection and escalation are terminal with no rollback needed.
- **View indexing (Lesson 2):** `ActionsView` exposes one query (`getAllActions`); status filtering happens client-side in the endpoint and monitor — enum columns cannot be auto-indexed.
