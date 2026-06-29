# PLAN — akka-stategraph-bridge

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams render on the generated system's Architecture tab.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Planner[PlannerAgent]:::agent
  Node[NodeAgent]:::agent
  Router[EdgeRouterAgent]:::agent

  WF[GraphWorkflow]:::wf
  Run[GraphRunEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[RunQueueEntity]:::ese
  View[GraphRunView]:::view
  Consumer[RunRequestConsumer]:::cons
  Sim[RunSimulator]:::ta
  Stuck[StuckRunMonitor]:::ta
  API[GraphEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN_GRAPH / REVISE_TOOL_BINDING| Planner
  WF -->|EXECUTE_NODE| Node
  WF -->|ROUTE_EDGE| Router
  WF -->|emit events| Run
  WF -->|poll| Ctrl
  Run -.->|projects| View
  API -->|query/SSE| View
  Stuck -.->|every 30s| Run
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as GraphEndpoint
  participant Q as RunQueueEntity
  participant C as RunRequestConsumer
  participant W as GraphWorkflow
  participant P as PlannerAgent
  participant N as NodeAgent
  participant R as EdgeRouterAgent
  participant E as GraphRunEntity
  participant CTL as SystemControlEntity
  participant V as GraphRunView

  U->>API: POST /api/runs {graphJson}
  API->>Q: append RunSubmitted
  API-->>U: 202 {runId}
  Q->>C: RunSubmitted
  C->>W: start({runId, graphJson})
  W->>E: emit RunCreated (PLANNING)
  W->>P: PLAN_GRAPH(graphJson)
  P-->>W: GraphPlan
  W->>E: emit RunPlanned, status EXECUTING
  loop until terminal | fail | halt
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>W: resolve current node from GraphPlan
    W->>W: guardrail.vet(NodeDispatch)
    W->>N: EXECUTE_NODE(NodeDispatch)
    N-->>W: NodeOutput
    W->>W: SecretScrubber.scrub(rawContent)
    W->>E: emit NodeExecuted (TraceEntry + state merge)
    W->>R: ROUTE_EDGE(nodeId, GraphState)
    R-->>W: RoutingDecision
    W->>W: checkCycleStep(nextNodeId)
  end
  W->>E: emit RunCompleted
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `GraphRunEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> EXECUTING: RunPlanned
  EXECUTING --> EXECUTING: NodeExecuted / NodeBlocked / StateUpdated
  EXECUTING --> COMPLETED: RunCompleted
  EXECUTING --> FAILED: RunFailed
  EXECUTING --> HALTED: RunHaltedOperator / RunHaltedAutomatic
  EXECUTING --> STUCK: RunFailedTimeout
  STUCK --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  GraphRunEntity ||--o{ RunPlanned : emits
  GraphRunEntity ||--o{ NodeDispatched : emits
  GraphRunEntity ||--o{ NodeBlocked : emits
  GraphRunEntity ||--o{ NodeExecuted : emits
  GraphRunEntity ||--o{ StateUpdated : emits
  GraphRunEntity ||--o{ RunCompleted : emits
  GraphRunEntity ||--o{ RunFailed : emits
  GraphRunEntity ||--o{ RunHaltedOperator : emits
  GraphRunEntity ||--o{ RunHaltedAutomatic : emits
  GraphRunEntity ||--o{ RunFailedTimeout : emits
  GraphRunView }o--|| GraphRunEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  RunQueueEntity ||--o{ RunSubmitted : emits
  RunRequestConsumer }o--|| RunQueueEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `NodeAgent` | `application/NodeAgent.java` |
| `EdgeRouterAgent` | `application/EdgeRouterAgent.java` |
| `GraphWorkflow` | `application/GraphWorkflow.java` |
| `GraphRunEntity` | `application/GraphRunEntity.java` (state in `domain/GraphRun.java`, events in `domain/GraphRunEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `RunQueueEntity` | `application/RunQueueEntity.java` |
| `GraphRunView` | `application/GraphRunView.java` |
| `RunRequestConsumer` | `application/RunRequestConsumer.java` |
| `RunSimulator` | `application/RunSimulator.java` |
| `StuckRunMonitor` | `application/StuckRunMonitor.java` |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `SecretScrubber` | `application/SecretScrubber.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `NodeTasks` | `application/NodeTasks.java` |
| `GraphEndpoint` | `api/GraphEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `executeNodeStep` 120 s (covers agent call + fixture lookup), `routeStep` 45 s, `completeStep` 60 s. Default recovery: `maxRetries(2).failoverTo(GraphWorkflow::error)`.
- **Cycle budget:** each `NodeDef.maxVisits` is declared in the graph definition (default 3). `checkCycleStep` increments a per-node visit map held in workflow-local state; exceeding the limit transitions to `failStep`.
- **Guardrail revision budget:** the planner may attempt `REVISE_TOOL_BINDING` at most twice per blocked node; a third block on the same node transitions to `failStep`.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during `executeNodeStep` lets the in-flight node finish; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `GraphEndpoint.submit` deduplicates `POST /api/runs` by `(graphJson hash, requestedBy)` over a 10 s window.
- **Stuck detection:** `StuckRunMonitor` ticks every 30 s; `RunFailedTimeout` is non-fatal to other runs. The workflow's `decideStep` checks entity status and exits when it reads `STUCK`.
- **Sanitizer determinism:** `SecretScrubber.scrub` is pure and stateless; identical input always yields identical scrubbed output, keeping `NodeExecuted` events deterministic and replayable.
