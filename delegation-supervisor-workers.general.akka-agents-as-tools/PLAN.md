# PLAN — OpenAI Agents-as-Tools Composition

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  CE[CompositionEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  TQ[TaskRequestQueue<br/>EventSourcedEntity]:::ese
  TC[TaskRequestConsumer<br/>Consumer]:::con
  WF[CompositionWorkflow<br/>Workflow]:::wf
  SV[Supervisor<br/>AutonomousAgent]:::ag
  SM[SummarizerAgent<br/>AutonomousAgent]:::ag
  CL[ClassifierAgent<br/>AutonomousAgent]:::ag
  TR[TranslatorAgent<br/>AutonomousAgent]:::ag
  TE[TaskRequestEntity<br/>EventSourcedEntity]:::ese
  VW[CompositionView<br/>View]:::vw
  SIM[RequestSimulator<br/>TimedAction]:::ta
  EV[EvalSampler<br/>TimedAction]:::ta

  CE -->|POST /tasks| TQ
  SIM -.->|every 60s| TQ
  TQ -.->|TaskSubmitted| TC
  TC -->|start workflow| WF
  WF -->|ROUTE| SV
  WF -->|guardrail check| WF
  WF -->|SUMMARIZE| SM
  WF -->|CLASSIFY| CL
  WF -->|TRANSLATE| TR
  WF -->|ASSEMBLE| SV
  WF -->|commands| TE
  TE -.->|events| VW
  EV -.->|every 5m| VW
  EV -->|recordEval| TE
  CE -->|getAllRequests / SSE| VW
  AE --> STATIC[static-resources]:::static

  classDef ep fill:#141414,stroke:#7EC8E3,color:#fff;
  classDef ese fill:#141414,stroke:#F5C518,color:#fff;
  classDef vw fill:#141414,stroke:#3fb950,color:#fff;
  classDef wf fill:#141414,stroke:#ff5f57,color:#fff;
  classDef ag fill:#141414,stroke:#B388FF,color:#fff;
  classDef con fill:#141414,stroke:#7EC8E3,color:#fff;
  classDef ta fill:#141414,stroke:#F5C518,color:#fff;
  classDef static fill:#0A0A0A,stroke:#333,color:#aaa;
```

Solid arrows: synchronous commands. Dashed arrows: event subscriptions. Dotted arrows: scheduled ticks.

## Interaction sequence

```mermaid
sequenceDiagram
  participant U as User / Simulator
  participant CE as CompositionEndpoint
  participant TQ as TaskRequestQueue
  participant WF as CompositionWorkflow
  participant SV as Supervisor
  participant SUB as Selected Sub-Agent
  participant TE as TaskRequestEntity

  U->>CE: POST /api/tasks {inputText, operation}
  CE->>TQ: enqueueTask
  TQ-->>WF: TaskRequestConsumer starts workflow
  WF->>TE: createRequest (QUEUED)
  WF->>SV: ROUTE -> RoutingDecision
  WF->>TE: attachRouting (IN_PROGRESS)
  WF->>WF: guardrailStep (before-tool-call)
  alt guardrail blocks
    WF->>TE: block (BLOCKED)
  else guardrail passes
    WF->>SUB: dispatch selected tool (SUMMARIZE / CLASSIFY / TRANSLATE)
    Note over WF: 60s stepTimeout; on timeout -> RequestTimedOut
    WF->>TE: attachToolResult
    WF->>SV: ASSEMBLE(routing, toolResult) -> TaskResult
    WF->>TE: complete (COMPLETED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> QUEUED
  QUEUED --> IN_PROGRESS: RoutingDecision ready
  IN_PROGRESS --> COMPLETED: guardrail pass + tool result assembled
  IN_PROGRESS --> BLOCKED: guardrail rejected tool call
  IN_PROGRESS --> TIMED_OUT: sub-agent step timeout
  BLOCKED --> [*]
  TIMED_OUT --> [*]
  COMPLETED --> COMPLETED: TaskEvalScored
  COMPLETED --> [*]
```

## Entity model

```mermaid
erDiagram
  TASK_REQUEST ||--o| ROUTING_DECISION : has
  TASK_REQUEST ||--o| TOOL_RESULT : has
  TASK_REQUEST ||--o| TASK_RESULT : produces
  TASK_REQUEST_QUEUE ||--|| TASK_REQUEST : seeds
  TASK_REQUEST {
    string requestId
    string inputText
    string operation
    enum status
    int evalScore
    instant createdAt
  }
  TASK_REQUEST_QUEUE {
    string requestId
    string inputText
    string operation
    string requestedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `Supervisor` | AutonomousAgent | `application/Supervisor.java` |
| `SummarizerAgent` | AutonomousAgent | `application/SummarizerAgent.java` |
| `ClassifierAgent` | AutonomousAgent | `application/ClassifierAgent.java` |
| `TranslatorAgent` | AutonomousAgent | `application/TranslatorAgent.java` |
| `CompositionTasks` | Task constants | `application/CompositionTasks.java` |
| `CompositionWorkflow` | Workflow | `application/CompositionWorkflow.java` |
| `TaskRequestEntity` | EventSourcedEntity | `domain/TaskRequestEntity.java` |
| `TaskRequestQueue` | EventSourcedEntity | `domain/TaskRequestQueue.java` |
| `CompositionView` | View | `application/CompositionView.java` |
| `TaskRequestConsumer` | Consumer | `application/TaskRequestConsumer.java` |
| `RequestSimulator` | TimedAction | `application/RequestSimulator.java` |
| `EvalSampler` | TimedAction | `application/EvalSampler.java` |
| `CompositionEndpoint` | HttpEndpoint | `api/CompositionEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `dispatchStep` and `assembleStep` each get 60s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Guardrail placement:** the guardrail step runs after `routeStep` returns a `RoutingDecision` but before the sub-agent is dispatched. This is the before-tool-call contract — the decision exists, but the tool has not been called yet.
- **Tool dispatch branching:** `dispatchStep` reads `RoutingDecision.selectedTool` and calls the corresponding agent. All three branches share the same 60s stepTimeout.
- **Idempotency:** the workflow id is the `requestId`. Re-delivery of the same `TaskSubmitted` event resolves to the same workflow instance — no duplicate request.
- **Eval sampling:** `EvalSampler` reads `CompositionView.getAllRequests` (no enum WHERE clause) and filters client-side for the oldest `COMPLETED` request lacking an `evalScore`.
