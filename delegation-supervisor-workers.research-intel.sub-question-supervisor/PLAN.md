# PLAN — Sub-Question Query Engine

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  QE[QueryEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  IQ[IndexCallQueue<br/>EventSourcedEntity]:::ese
  IC[IndexCallConsumer<br/>Consumer]:::con
  WF[QueryOrchestrationWorkflow<br/>Workflow]:::wf
  QS[QuerySupervisor<br/>AutonomousAgent]:::ag
  IW[IndexWorker<br/>AutonomousAgent]:::ag
  SE[QuerySessionEntity<br/>EventSourcedEntity]:::ese
  VW[QuerySessionView<br/>View]:::vw
  SIM[QuestionSimulator<br/>TimedAction]:::ta
  EV[EvalSampler<br/>TimedAction]:::ta

  QE -->|POST /query| IQ
  SIM -.->|every 60s| IQ
  IQ -.->|QuestionSubmitted| IC
  IC -->|start workflow| WF
  WF -->|DECOMPOSE| QS
  WF -->|RETRIEVE × N| IW
  WF -->|SYNTHESISE| QS
  WF -->|commands| SE
  SE -.->|events| VW
  EV -.->|every 5m| VW
  EV -->|recordEval| SE
  QE -->|getAllSessions / SSE| VW
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
  participant QE as QueryEndpoint
  participant IQ as IndexCallQueue
  participant WF as QueryOrchestrationWorkflow
  participant QS as QuerySupervisor
  participant IW as IndexWorker
  participant SE as QuerySessionEntity

  U->>QE: POST /api/query {question}
  QE->>IQ: enqueueQuestion
  IQ-->>WF: IndexCallConsumer starts workflow
  WF->>SE: createSession (DECOMPOSING)
  WF->>QS: DECOMPOSE -> DecompositionPlan
  WF->>SE: status RETRIEVING
  par parallel sub-question dispatch
    WF->>IW: RETRIEVE (subQuestion_1) [guardrail checks before index call]
  and
    WF->>IW: RETRIEVE (subQuestion_2) [guardrail checks before index call]
  and
    WF->>IW: RETRIEVE (subQuestion_N) [...]
  end
  Note over WF: join; if any step times out (60s) -> partialStep
  WF->>QS: SYNTHESISE(indexResults) -> CombinedAnswer
  WF->>SE: synthesise (SYNTHESISED)
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> DECOMPOSING
  DECOMPOSING --> RETRIEVING: DecompositionPlan ready
  RETRIEVING --> SYNTHESISED: all results collected + synthesis
  RETRIEVING --> PARTIAL: one or more workers timed out
  RETRIEVING --> BLOCKED: all sub-questions rejected by guardrail
  PARTIAL --> [*]
  BLOCKED --> [*]
  SYNTHESISED --> SYNTHESISED: SynthesisEvalScored
  SYNTHESISED --> [*]
```

## Entity model

```mermaid
erDiagram
  QUERY_SESSION ||--o| DECOMPOSITION_PLAN : has
  QUERY_SESSION ||--o{ INDEX_RESULT : collects
  QUERY_SESSION ||--o| COMBINED_ANSWER : produces
  INDEX_CALL_QUEUE ||--|| QUERY_SESSION : seeds
  QUERY_SESSION {
    string sessionId
    string question
    enum status
    int evalScore
    instant createdAt
  }
  INDEX_CALL_QUEUE {
    string sessionId
    string question
    string requestedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `QuerySupervisor` | AutonomousAgent | `application/QuerySupervisor.java` |
| `IndexWorker` | AutonomousAgent | `application/IndexWorker.java` |
| `QueryTasks` | Task constants | `application/QueryTasks.java` |
| `QueryOrchestrationWorkflow` | Workflow | `application/QueryOrchestrationWorkflow.java` |
| `QuerySessionEntity` | EventSourcedEntity | `domain/QuerySessionEntity.java` |
| `IndexCallQueue` | EventSourcedEntity | `domain/IndexCallQueue.java` |
| `QuerySessionView` | View | `application/QuerySessionView.java` |
| `IndexCallConsumer` | Consumer | `application/IndexCallConsumer.java` |
| `QuestionSimulator` | TimedAction | `application/QuestionSimulator.java` |
| `EvalSampler` | TimedAction | `application/EvalSampler.java` |
| `QueryEndpoint` | HttpEndpoint | `api/QueryEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** each `retrieveStep` (one per sub-question) gets 60s; `synthesiseStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** all retrieval steps run concurrently via `CompletionStage` zip over the full list of sub-questions. Not sequential.
- **Guardrail placement:** the before-tool-call check runs inside `IndexWorker` before the seeded index tool is called. A `GuardrailException` propagates back to the workflow, which records an `IndexCallRejected` event and skips that sub-question.
- **Partial path (compensation):** if any worker times out, `defaultStepRecovery` routes to `partialStep`, which synthesises from whichever results arrived and ends with `SessionPartial`. No infinite retry.
- **All-rejected path:** if every sub-question is rejected by the guardrail, the workflow transitions to `SessionBlocked` with a `failureReason` listing the rejected sub-questions.
- **Idempotency:** the workflow id is the `sessionId`. Re-delivery of the same `QuestionSubmitted` event resolves to the same workflow instance — no duplicate session.
- **Eval sampling:** `EvalSampler` reads `QuerySessionView.getAllSessions` (no enum WHERE clause) and filters client-side for the oldest `SYNTHESISED` session lacking an `evalScore`.
