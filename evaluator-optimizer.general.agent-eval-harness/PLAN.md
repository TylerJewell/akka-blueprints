# PLAN — agent-eval-harness

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab.

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

  Evaluator[EvaluatorAgent]:::agent
  Judge[JudgeAgent]:::agent

  WF[EvalRunWorkflow]:::wf
  Run[EvalRunEntity]:::ese
  Suite[SuiteRegistry]:::ese
  View[EvalRunsView]:::view
  Consumer[SuiteEventConsumer]:::cons
  Scheduler[RunScheduler]:::ta
  Sampler[AccuracySampler]:::ta
  API[EvalEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|register suite / trigger run| Suite
  Scheduler -.->|every 5 min| Suite
  Suite -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|invoke case| Evaluator
  WF -->|judge response| Judge
  WF -->|emit events| Run
  Run -.->|projects| View
  API -->|query / SSE| View
  Sampler -.->|every 60 s| Run
```

## Interaction sequence — J1 (passing run, 3 cases)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as EvalEndpoint
  participant S as SuiteRegistry
  participant C as SuiteEventConsumer
  participant W as EvalRunWorkflow
  participant EV as EvaluatorAgent
  participant JG as JudgeAgent
  participant R as EvalRunEntity
  participant V as EvalRunsView

  U->>API: POST /api/runs {suiteName, passingThreshold}
  API->>S: triggerRun
  API-->>U: 202 {runId}
  S->>C: RunTriggered
  C->>W: start({runId, suiteName, threshold})
  W->>R: emit RunCreated (PENDING)
  W->>R: emit RunStarted (RUNNING)

  W->>EV: INVOKE_CASE(case 1 input)
  EV-->>W: CaseResponse{rawOutput}
  W->>JG: JUDGE_RESPONSE(response, expected, rubric)
  JG-->>W: Judgment{PASS, score=0.9}
  W->>R: emit CaseEvaluated (case 1, PASS)

  W->>EV: INVOKE_CASE(case 2 input)
  EV-->>W: CaseResponse{rawOutput}
  W->>JG: JUDGE_RESPONSE(response, expected, rubric)
  JG-->>W: Judgment{PASS, score=0.85}
  W->>R: emit CaseEvaluated (case 2, PASS)

  W->>EV: INVOKE_CASE(case 3 input)
  EV-->>W: CaseResponse{rawOutput}
  W->>JG: JUDGE_RESPONSE(response, expected, rubric)
  JG-->>W: Judgment{PASS, score=0.92}
  W->>R: emit CaseEvaluated (case 3, PASS)

  Note over W: aggregateStep — passRate = 3/3 = 1.0 >= 0.8
  W->>R: emit RunPassed (passRate=1.0)
  R-->>V: project
  V-->>U: SSE update
```

## State machine — `EvalRunEntity`

```mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> RUNNING: RunStarted
  RUNNING --> RUNNING: CaseEvaluated (more cases remain)
  RUNNING --> PASSED: RunPassed (passRate >= threshold)
  RUNNING --> FAILED: GateFailed + RunFailed (passRate < threshold)
  RUNNING --> FAILED: defaultStepRecovery failover
  PASSED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  EvalRunEntity ||--o{ RunCreated : emits
  EvalRunEntity ||--o{ RunStarted : emits
  EvalRunEntity ||--o{ CaseEvaluated : emits
  EvalRunEntity ||--o{ GateFailed : emits
  EvalRunEntity ||--o{ RunPassed : emits
  EvalRunEntity ||--o{ RunFailed : emits
  EvalRunEntity ||--o{ AccuracySnapshot : emits
  EvalRunsView }o--|| EvalRunEntity : projects
  SuiteRegistry ||--o{ SuiteRegistered : emits
  SuiteRegistry ||--o{ RunTriggered : emits
  SuiteEventConsumer }o--|| SuiteRegistry : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `EvaluatorAgent` | `application/EvaluatorAgent.java` |
| `JudgeAgent` | `application/JudgeAgent.java` |
| `EvalTasks` | `application/EvalTasks.java` |
| `EvalRunWorkflow` | `application/EvalRunWorkflow.java` |
| `EvalRunEntity` | `application/EvalRunEntity.java` (state in `domain/EvalRun.java`, events in `domain/EvalRunEvent.java`) |
| `SuiteRegistry` | `application/SuiteRegistry.java` |
| `EvalRunsView` | `application/EvalRunsView.java` |
| `SuiteEventConsumer` | `application/SuiteEventConsumer.java` |
| `RunScheduler` | `application/RunScheduler.java` |
| `AccuracySampler` | `application/AccuracySampler.java` |
| `EvalEndpoint` | `api/EvalEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `invokeStep` and `judgeStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(failStep))` — the workflow degrades to `FAILED` on irrecoverable agent failure rather than hanging.
- **Sequential case processing:** the workflow processes cases one at a time, writing a `CaseEvaluated` event before moving to the next case. This keeps the entity state consistent and lets the SSE stream reflect incremental progress.
- **AccuracySampler idempotency:** the sampler keys its `recordAccuracySnapshot` calls on `runId` so a tick that fires twice for the same completed run is a no-op on the entity side.
- **Aggregate step:** `aggregateStep` is pure-function (no LLM call); it counts `PASS` results from the accumulated `results` list and computes `passRate`. No external I/O.
- **Saga semantics:** there is no external side-effect to compensate. The CI gate (`G1`) is the only "compensation path"; it records a `GateFailed` event before transitioning to `FAILED` so the gate decision is always observable.
- **Suite isolation:** each run is keyed by `runId`; multiple concurrent runs on different suites do not share workflow or entity state.
