# PLAN — Inspect Multi-Agent Run (Phoenix)

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  RE[RunEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  QQ[QueryQueue<br/>EventSourcedEntity]:::ese
  RC[RunRequestConsumer<br/>Consumer]:::con
  WF[RunWorkflow<br/>Workflow]:::wf
  MA[ManagerAgent<br/>AutonomousAgent]:::ag
  SA[SearchAgent<br/>AutonomousAgent]:::ag
  TI[TraceInspector<br/>AutonomousAgent]:::ag
  RN[RunEntity<br/>EventSourcedEntity]:::ese
  VW[RunView<br/>View]:::vw
  SIM[QuerySimulator<br/>TimedAction]:::ta
  EV[EvalSampler<br/>TimedAction]:::ta

  RE -->|POST /runs| QQ
  SIM -.->|every 60s| QQ
  QQ -.->|QuerySubmitted| RC
  RC -->|start workflow| WF
  WF -->|PLAN_QUERY| MA
  WF -->|EXECUTE_SEARCH| SA
  WF -->|INSPECT_TRACE| TI
  WF -->|SYNTHESISE_ANSWER| MA
  WF -->|commands| RN
  RN -.->|events| VW
  EV -.->|every 5m| VW
  EV -->|recordEval| RN
  RE -->|getAllRuns / SSE| VW
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
  participant RE as RunEndpoint
  participant QQ as QueryQueue
  participant WF as RunWorkflow
  participant MA as ManagerAgent
  participant SA as SearchAgent
  participant TI as TraceInspector
  participant RN as RunEntity

  U->>RE: POST /api/runs {query}
  RE->>QQ: enqueueQuery
  QQ-->>WF: RunRequestConsumer starts workflow
  WF->>RN: createRun (DISPATCHED)
  WF->>MA: PLAN_QUERY -> SearchPlan
  WF->>RN: recordPlan
  WF->>RN: status SEARCHING
  WF->>SA: EXECUTE_SEARCH -> SearchResultBundle
  Note over SA: before-tool-call guardrail checks each URL
  alt URL blocked
    WF->>RN: blockTool (BLOCKED_TOOL)
  else search completed
    WF->>RN: recordSearch
    WF->>WF: assembleTraceStep (builds RunTrace from spans)
    WF->>RN: recordTrace
    WF->>TI: INSPECT_TRACE -> TraceReport
    WF->>RN: recordTraceReport
    WF->>MA: SYNTHESISE_ANSWER -> SynthesisedAnswer
    WF->>RN: synthesise (TRACED)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> DISPATCHED
  DISPATCHED --> SEARCHING: SearchPlan ready
  SEARCHING --> TRACED: search + trace + synthesis complete
  SEARCHING --> PARTIAL: maxPages exceeded
  SEARCHING --> BLOCKED_TOOL: URL allow-list violation
  PARTIAL --> [*]
  BLOCKED_TOOL --> [*]
  TRACED --> TRACED: RunEvalScored
  TRACED --> [*]
```

## Entity model

```mermaid
erDiagram
  AGENT_RUN ||--o| SEARCH_PLAN : has
  AGENT_RUN ||--o| SEARCH_RESULT_BUNDLE : has
  AGENT_RUN ||--o| RUN_TRACE : has
  AGENT_RUN ||--o| TRACE_REPORT : has
  AGENT_RUN ||--o| SYNTHESISED_ANSWER : produces
  QUERY_QUEUE ||--|| AGENT_RUN : seeds
  AGENT_RUN {
    string runId
    string query
    enum status
    int evalScore
    instant createdAt
  }
  QUERY_QUEUE {
    string runId
    string query
    string requestedBy
    instant submittedAt
  }
  RUN_TRACE {
    int totalTokens
    long totalDurationMs
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `ManagerAgent` | AutonomousAgent | `application/ManagerAgent.java` |
| `SearchAgent` | AutonomousAgent | `application/SearchAgent.java` |
| `TraceInspector` | AutonomousAgent | `application/TraceInspector.java` |
| `RunTasks` | Task constants | `application/RunTasks.java` |
| `RunWorkflow` | Workflow | `application/RunWorkflow.java` |
| `RunEntity` | EventSourcedEntity | `domain/RunEntity.java` |
| `QueryQueue` | EventSourcedEntity | `domain/QueryQueue.java` |
| `RunView` | View | `application/RunView.java` |
| `RunRequestConsumer` | Consumer | `application/RunRequestConsumer.java` |
| `QuerySimulator` | TimedAction | `application/QuerySimulator.java` |
| `EvalSampler` | TimedAction | `application/EvalSampler.java` |
| `RunEndpoint` | HttpEndpoint | `api/RunEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `searchStep` and `synthesiseStep` get 90s; `inspectStep` gets 60s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Guard before tool calls:** the before-tool-call guardrail runs synchronously inside `SearchAgent`'s tool dispatch before any HTTP request is made. The allow-list is a config-loaded `Set<String>`.
- **Trace assembly:** `assembleTraceStep` is a deterministic Java step — it collects `PhoenixSpan` records emitted during `searchStep` and sums token counts. No LLM call, no timeout risk.
- **Idempotency:** the workflow id is the `runId`. Re-delivery of the same `QuerySubmitted` event resolves to the same workflow instance — no duplicate runs.
- **Partial path:** if `SearchAgent` signals it has exceeded `maxPages`, the workflow transitions to `partialStep` which synthesises from available results and ends with `RunPartial`.
- **Eval sampling:** `EvalSampler` reads `RunView.getAllRuns` (no enum WHERE clause) and filters client-side for the oldest `TRACED` run lacking an `evalScore`.
