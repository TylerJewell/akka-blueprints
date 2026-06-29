# PLAN — trajectory-eval

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

  Runner[RunnerAgent]:::agent
  Evaluator[EvaluatorAgent]:::agent

  WF[TrajectoryWorkflow]:::wf
  Eval[EvaluationEntity]:::ese
  Ref[ReferenceStore]:::ese
  View[EvaluationsView]:::view
  Consumer[ScenarioConsumer]:::cons
  Sim[ScenarioSimulator]:::ta
  Sampler[EvalSampler]:::ta
  API[TrajectoryEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit scenario| Ref
  API -->|start evaluation| WF
  Sim -.->|every 60s| API
  Ref -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|execute / re-execute| Runner
  WF -->|evaluate| Evaluator
  WF -->|emit events| Eval
  Eval -.->|projects| View
  API -->|query / SSE| View
  Sampler -.->|every 30s| Eval
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as TrajectoryEndpoint
  participant Ref as ReferenceStore
  participant C as ScenarioConsumer
  participant W as TrajectoryWorkflow
  participant R as RunnerAgent
  participant EV as EvaluatorAgent
  participant E as EvaluationEntity
  participant V as EvaluationsView

  U->>API: POST /api/evaluations {scenarioId}
  API->>Ref: getReference(scenarioId)
  Ref-->>API: ReferencePath (6 steps)
  API->>E: emit EvaluationCreated (RUNNING)
  API-->>U: 202 {evaluationId}
  E->>C: EvaluationCreated
  C->>W: start({evaluationId, scenarioId, taskDescription, maxAttempts=4})

  W->>R: EXECUTE(scenarioId, taskDescription)
  R-->>W: RecordedTrajectory #1 (8 steps)
  W->>E: emit TrajectoryRecorded (n=1)
  Note over W: guardrailStep (deterministic step-count check)
  W->>E: emit TrajectoryGuardrailVerdictRecorded (passed=true)
  W->>E: status EVALUATING
  W->>EV: EVALUATE(trajectory #1, referencePath)
  EV-->>W: TrajectoryVerdict{FAIL, deviationCount=2, report}
  W->>E: emit TrajectoryEvaluated (n=1, FAIL)

  W->>R: RE_EXECUTE(scenarioId, taskDescription, referencePath, deviationReport)
  R-->>W: RecordedTrajectory #2 (6 steps)
  W->>E: emit TrajectoryRecorded (n=2)
  W->>E: emit TrajectoryGuardrailVerdictRecorded (passed=true)
  W->>EV: EVALUATE(trajectory #2, referencePath)
  EV-->>W: TrajectoryVerdict{PASS, deviationCount=0, rationale}
  W->>E: emit TrajectoryEvaluated (n=2, PASS)
  W->>E: emit EvaluationPassed (n=2)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `EvaluationEntity`

```mermaid
stateDiagram-v2
  [*] --> RUNNING
  RUNNING --> EVALUATING: TrajectoryRecorded + guardrail passed
  RUNNING --> RUNNING: guardrail blocked, re-execute
  EVALUATING --> RUNNING: Verdict = FAIL, attempts < max
  EVALUATING --> PASSED: Verdict = PASS
  EVALUATING --> FAILED_FINAL: Verdict = FAIL, attempts = max
  PASSED --> [*]
  FAILED_FINAL --> [*]
```

## Entity model

```mermaid
erDiagram
  EvaluationEntity ||--o{ EvaluationCreated : emits
  EvaluationEntity ||--o{ TrajectoryRecorded : emits
  EvaluationEntity ||--o{ TrajectoryGuardrailVerdictRecorded : emits
  EvaluationEntity ||--o{ TrajectoryEvaluated : emits
  EvaluationEntity ||--o{ EvaluationPassed : emits
  EvaluationEntity ||--o{ EvaluationFailedFinal : emits
  EvaluationEntity ||--o{ EvalRecorded : emits
  EvaluationsView }o--|| EvaluationEntity : projects
  ReferenceStore ||--o{ ReferenceRegistered : emits
  ReferenceStore ||--o{ ReferenceUpdated : emits
  ScenarioConsumer }o--|| EvaluationEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RunnerAgent` | `application/RunnerAgent.java` |
| `EvaluatorAgent` | `application/EvaluatorAgent.java` |
| `TrajectoryTasks` | `application/TrajectoryTasks.java` |
| `TrajectoryWorkflow` | `application/TrajectoryWorkflow.java` |
| `EvaluationEntity` | `application/EvaluationEntity.java` (state in `domain/Evaluation.java`, events in `domain/EvaluationEvent.java`) |
| `ReferenceStore` | `application/ReferenceStore.java` |
| `EvaluationsView` | `application/EvaluationsView.java` |
| `ScenarioConsumer` | `application/ScenarioConsumer.java` |
| `ScenarioSimulator` | `application/ScenarioSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `TrajectoryEndpoint` | `api/TrajectoryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `executeStep` and `evaluateStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(failStep))` — the workflow degrades to `FAILED_FINAL` on irrecoverable agent failure rather than hanging.
- **Idempotency:** `TrajectoryEndpoint.submit` uses `(scenarioId, requestedBy)` over a 10 s window as the dedup key.
- **EvalSampler idempotency:** the sampler keys its `recordEval` calls on `(evaluationId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op on the entity side.
- **maxAttempts ceiling:** read from `trajectory-eval.runner.max-attempts` (default 4). The workflow checks the count BEFORE calling `executeStep` for the next iteration; it never recurses past the ceiling.
- **Saga semantics:** there is no external side-effect to compensate. The halt mechanism (`HT1`) is the only "compensation"; it preserves the closest-match trajectory and every deviation report on the entity.
- **Guardrail step:** `guardrailStep` is pure-function (no LLM call); it computes the step count from the trajectory and either advances to `evaluateStep` or returns to `executeStep` with a structured feedback note. The feedback note is a deterministic `DeviationReport` payload; it is never LLM-generated.
- **Reference path lookup:** the workflow fetches the `ReferencePath` from `ReferenceStore` at `startStep` and passes it into every subsequent `evaluateStep`. It does not re-fetch mid-loop; reference updates take effect on the next evaluation submission.
