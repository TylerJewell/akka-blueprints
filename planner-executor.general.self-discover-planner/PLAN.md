# PLAN — self-discover-planner

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
  Eval[PlanEvaluatorAgent]:::agent
  Exec[ExecutorAgent]:::agent
  Synth[SynthesiserAgent]:::agent

  WF[SolveWorkflow]:::wf
  Solve[SolveEntity]:::ese
  Registry[ModuleRegistry]:::ese
  Queue[RequestQueue]:::ese
  View[SolveView]:::view
  Consumer[SolveRequestConsumer]:::cons
  Sim[RequestSimulator]:::ta
  Stuck[StuckSolveMonitor]:::ta
  API[SolveEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|query registry| Registry
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|COMPOSE_PLAN / REVISE_PLAN| Planner
  WF -->|EVALUATE_PLAN| Eval
  WF -->|EXECUTE_MODULE| Exec
  WF -->|SYNTHESISE| Synth
  WF -->|emit events| Solve
  WF -->|read modules| Registry
  Solve -.->|projects| View
  API -->|query/SSE| View
  Stuck -.->|every 30s| Solve
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as SolveEndpoint
  participant Q as RequestQueue
  participant C as SolveRequestConsumer
  participant W as SolveWorkflow
  participant P as PlannerAgent
  participant EV as PlanEvaluatorAgent
  participant EX as ExecutorAgent
  participant S as SynthesiserAgent
  participant E as SolveEntity
  participant V as SolveView

  U->>API: POST /api/solves {prompt}
  API->>Q: append SolveSubmitted
  API-->>U: 202 {solveId}
  Q->>C: SolveSubmitted
  C->>W: start({solveId, prompt})
  W->>E: emit SolveCreated (PLANNING)
  W->>P: COMPOSE_PLAN(prompt, modules)
  P-->>W: ReasoningPlan
  W->>E: emit SolvePlanned, status EVALUATING
  W->>EV: EVALUATE_PLAN(plan)
  EV-->>W: PlanEval{verdict=PASS, score=0.82}
  W->>E: emit PlanAccepted, status EXECUTING
  loop for each step in stepOrder
    W->>EX: EXECUTE_MODULE(step, executionLog)
    EX-->>W: ModuleResult
    W->>W: OutputScrubber.scrub(result.output)
    W->>E: emit ModuleExecuted
  end
  W->>S: SYNTHESISE(executionLog)
  S-->>W: TaskAnswer
  W->>E: emit SolveCompleted (COMPLETED)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `SolveEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> EVALUATING: SolvePlanned
  EVALUATING --> EVALUATING: PlanRejected (revision loop)
  EVALUATING --> EXECUTING: PlanAccepted
  EXECUTING --> EXECUTING: ModuleExecuted
  EXECUTING --> COMPLETED: SolveCompleted
  EXECUTING --> FAILED: SolveFailed
  EXECUTING --> STUCK: SolveFailedTimeout
  COMPLETED --> [*]
  FAILED --> [*]
  STUCK --> [*]
```

## Entity model

```mermaid
erDiagram
  SolveEntity ||--o{ SolveCreated : emits
  SolveEntity ||--o{ SolvePlanned : emits
  SolveEntity ||--o{ PlanAccepted : emits
  SolveEntity ||--o{ PlanRejected : emits
  SolveEntity ||--o{ ModuleExecuted : emits
  SolveEntity ||--o{ SolveCompleted : emits
  SolveEntity ||--o{ SolveFailed : emits
  SolveEntity ||--o{ SolveFailedTimeout : emits
  SolveView }o--|| SolveEntity : projects
  ModuleRegistry ||--o{ RegistrySeeded : emits
  RequestQueue ||--o{ SolveSubmitted : emits
  SolveRequestConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `PlanEvaluatorAgent` | `application/PlanEvaluatorAgent.java` |
| `ExecutorAgent` | `application/ExecutorAgent.java` |
| `SynthesiserAgent` | `application/SynthesiserAgent.java` |
| `SolveWorkflow` | `application/SolveWorkflow.java` |
| `SolveEntity` | `application/SolveEntity.java` (state in `domain/Solve.java`, events in `domain/SolveEvent.java`) |
| `ModuleRegistry` | `application/ModuleRegistry.java` |
| `RequestQueue` | `application/RequestQueue.java` |
| `SolveView` | `application/SolveView.java` |
| `SolveRequestConsumer` | `application/SolveRequestConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `StuckSolveMonitor` | `application/StuckSolveMonitor.java` |
| `OutputScrubber` | `application/OutputScrubber.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `EvaluatorTasks` | `application/EvaluatorTasks.java` |
| `ExecutorTasks` | `application/ExecutorTasks.java` |
| `SynthesiserTasks` | `application/SynthesiserTasks.java` |
| `SolveEndpoint` | `api/SolveEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `composePlanStep` 60 s, `revisePlanStep` 60 s, `evaluatePlanStep` 45 s, `executeModuleStep` 90 s (per module call), `synthesiseStep` 60 s, `completeStep` 30 s. Default recovery: `maxRetries(2).failoverTo(SolveWorkflow::error)`.
- **Revision budget:** the evaluator may return `FAIL` at most twice. A third rejection transitions the workflow to `failStep` with a `failureReason` of `"plan rejected after maximum revisions"`.
- **Module loop:** `executeModuleStep` iterates the `stepOrder` list sequentially; each iteration is a separate workflow step call. The `ExecutionLog` passed to each `ExecutorAgent` call contains all previously recorded steps so the agent can use prior outputs as inputs.
- **Idempotency:** `SolveEndpoint.submit` deduplicates `POST /api/solves` on `(prompt, requestedBy)` over a 10 s window.
- **Stuck detection:** `StuckSolveMonitor` ticks every 30 s; `SolveFailedTimeout` is non-fatal to other solves.
- **Scrubber determinism:** `OutputScrubber.scrub` is pure. The same input always yields the same scrubbed output, keeping `ModuleExecuted` events deterministic and replayable.
- **Registry read:** `SolveWorkflow.composePlanStep` reads `ModuleRegistry.getModules()` once and passes the module list as context to `PlannerAgent`. The registry is not polled mid-solve.
