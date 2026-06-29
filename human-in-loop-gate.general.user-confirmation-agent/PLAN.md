# PLAN — user-confirmation-agent

Architectural sketch for User Confirmation Agents. All four mermaid diagrams plus the component table.

---

## Component graph

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#0e1e2a", "edgeLabelBackground": "#1a1a1a", "transitionLabelColor": "#cccccc"}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  EP[ConfirmationEndpoint]:::ep
  APP[AppEndpoint]:::ep
  WF[ConfirmationWorkflow]:::wf
  PA[PlannerAgent]:::agent
  EA[ExecutorAgent]:::agent
  RE[RequestEntity]:::ese
  RV[RequestsView]:::view

  EP -->|confirmation-request| WF
  WF -->|plan task| PA
  WF -->|execute task| EA
  PA -->|recordPlan| RE
  EA -->|recordExecution| RE
  EP -->|confirm / cancel| RE
  WF -->|poll status| RE
  RE -.->|events| RV
  EP -->|getAllRequests / SSE| RV
  APP -->|static UI| EP
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor User
  participant EP as ConfirmationEndpoint
  participant WF as ConfirmationWorkflow
  participant PA as PlannerAgent
  participant RE as RequestEntity
  participant EA as ExecutorAgent

  User->>EP: POST /api/confirmation-request {description}
  EP->>WF: start(requestId, description)
  WF->>PA: runSingleTask(PLAN)
  PA-->>WF: ActionPlan{actions}
  WF->>RE: recordPlan -> PENDING_CONFIRMATION
  Note over WF,RE: await-confirmation task paused; workflow polls status every 5s
  User->>EP: POST /api/requests/{id}/confirm
  EP->>RE: confirm -> CONFIRMED
  WF->>RE: getRequest -> CONFIRMED
  WF->>EA: runSingleTask(EXECUTE) [guard: status == CONFIRMED]
  EA-->>WF: ExecutionResult{summary, executedAt}
  WF->>RE: recordExecution -> EXECUTED
```

## State machine

```mermaid
%%{init: {"theme": "base", "themeVariables": {"transitionLabelColor": "#cccccc"}}}%%
stateDiagram-v2
  [*] --> PENDING_CONFIRMATION: RequestPlanned
  PENDING_CONFIRMATION --> CONFIRMED: RequestConfirmed
  PENDING_CONFIRMATION --> CANCELLED: RequestCancelled
  CONFIRMED --> EXECUTED: RequestExecuted
  CANCELLED --> [*]
  EXECUTED --> [*]
```

## Entity model

```mermaid
erDiagram
  REQUEST ||--o{ REQUEST_EVENT : emits
  REQUEST {
    string id
    string description
    string status
    string proposedActions
    string executionSummary
    string executedAt
  }
  REQUEST_EVENT {
    string type
    string occurredAt
  }
  REQUESTS_VIEW {
    string id
    string status
  }
  REQUEST ||--|| REQUESTS_VIEW : projects
```

## Component table

| Component | Path (generated) |
|---|---|
| PlannerAgent | `application/PlannerAgent.java` |
| ExecutorAgent | `application/ExecutorAgent.java` |
| ConfirmationWorkflow | `application/ConfirmationWorkflow.java` |
| ConfirmationTasks | `application/ConfirmationTasks.java` |
| RequestEntity | `application/RequestEntity.java` |
| RequestsView | `application/RequestsView.java` |
| ConfirmationEndpoint | `api/ConfirmationEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| Request / events / records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `planStep` and `executeStep` call agents; both set `stepTimeout(60s)` to absorb LLM latency. The default 5 s step timeout would retry forever (Lesson 4).
- **Await-confirmation task.** The workflow does not block a thread; `awaitConfirmationStep` reads `RequestEntity.getRequest`, and on `PENDING_CONFIRMATION` self-schedules a 5-second resume timer until the human transitions the status.
- **Idempotency.** `requestId` is the workflow id and the entity id; re-delivery of `recordPlan` / `recordExecution` is absorbed by event-applier checks on current status.
- **Execution guard.** Before the execute tool runs, the before-tool-call guardrail re-reads `RequestEntity.status`; if it is not `CONFIRMED`, the call is blocked. No compensation path is needed because execution is the terminal write.
