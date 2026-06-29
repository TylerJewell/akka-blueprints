# PLAN — multi-agent-eval-harness

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

  Orch[OrchestratorAgent]:::agent
  Judge[JudgeAgent]:::agent

  WF[EvalRunWorkflow]:::wf
  Run[EvalRunEntity]:::ese
  Registry[ScenarioRegistry]:::ese
  View[EvalRunsView]:::view
  Consumer[ScenarioResultConsumer]:::cons
  Loader[ScenarioLoader]:::ta
  Sampler[EvalSampler]:::ta
  API[EvalEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|register suite| Registry
  Loader -.->|every 120s| Registry
  Registry -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|dispatch scenarios| Orch
  WF -->|judge results| Judge
  WF -->|emit events| Run
  Run -.->|projects| View
  API -->|query / SSE| View
  Sampler -.->|every 60s| Run
```

## Interaction sequence — J1 (passing run on attempt 1)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as EvalEndpoint
  participant R as ScenarioRegistry
  participant C as ScenarioResultConsumer
  participant W as EvalRunWorkflow
  participant O as OrchestratorAgent
  participant J as JudgeAgent
  participant E as EvalRunEntity
  participant V as EvalRunsView

  U->>API: POST /api/runs {suiteName, passThreshold}
  API->>R: registerSuite (SuiteRegistered)
  API-->>U: 202 {runId}
  R->>C: SuiteRegistered
  C->>W: start({runId, suiteName, passThreshold, maxRetries=2})
  W->>E: emit RunCreated (RUNNING)

  W->>R: getSuite(suiteName)
  R-->>W: List<ScenarioSpec>
  W->>E: emit ScenariosLoaded

  W->>O: DISPATCH_SCENARIOS(scenarios)
  O-->>W: AggregateResult {aggregateScore=0.88}
  W->>E: emit OrchestratorResultRecorded
  Note over W: thresholdStep (deterministic score check — passes)
  W->>E: status JUDGING
  W->>J: JUDGE_RUN(AggregateResult)
  J-->>W: Judgment{PASS, score=0.90, rationale}
  W->>E: emit JudgmentRecorded (PASS)
  W->>E: emit RunPassed (score=0.90)
  Note over W: ciGateStep — score >= threshold, no block
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `EvalRunEntity`

```mermaid
stateDiagram-v2
  [*] --> RUNNING
  RUNNING --> JUDGING: OrchestratorResult received + threshold check passes
  RUNNING --> RUNNING: threshold check fails, retries remaining
  JUDGING --> RUNNING: Judgment = FAIL, retries remaining
  JUDGING --> PASSED: Judgment = PASS
  JUDGING --> FAILED: Judgment = FAIL, retries exhausted
  PASSED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  EvalRunEntity ||--o{ RunCreated : emits
  EvalRunEntity ||--o{ ScenariosLoaded : emits
  EvalRunEntity ||--o{ OrchestratorResultRecorded : emits
  EvalRunEntity ||--o{ JudgmentRecorded : emits
  EvalRunEntity ||--o{ RunPassed : emits
  EvalRunEntity ||--o{ RunFailed : emits
  EvalRunEntity ||--o{ CIGateBlocked : emits
  EvalRunEntity ||--o{ ScenarioEvalRecorded : emits
  EvalRunsView }o--|| EvalRunEntity : projects
  ScenarioRegistry ||--o{ SuiteRegistered : emits
  ScenarioResultConsumer }o--|| ScenarioRegistry : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `OrchestratorAgent` | `application/OrchestratorAgent.java` |
| `JudgeAgent` | `application/JudgeAgent.java` |
| `EvalTasks` | `application/EvalTasks.java` |
| `EvalRunWorkflow` | `application/EvalRunWorkflow.java` |
| `EvalRunEntity` | `application/EvalRunEntity.java` (state in `domain/EvalRun.java`, events in `domain/EvalRunEvent.java`) |
| `ScenarioRegistry` | `application/ScenarioRegistry.java` |
| `EvalRunsView` | `application/EvalRunsView.java` |
| `ScenarioResultConsumer` | `application/ScenarioResultConsumer.java` |
| `ScenarioLoader` | `application/ScenarioLoader.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `EvalEndpoint` | `api/EvalEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `dispatchStep` and `judgeStep` each carry `stepTimeout(Duration.ofSeconds(90))`. The orchestrator may fan out across multiple specialist paths; 90 s accommodates the aggregate latency. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(failStep))` — any unrecoverable agent failure ends in `FAILED` rather than a hung workflow.
- **Idempotency:** `EvalEndpoint.submit` deduplicates on `(suiteName, requestedBy)` over a 15 s window.
- **EvalSampler idempotency:** the sampler keys its `recordScenarioEval` calls on `(runId, scenarioId)` so a tick that fires twice for the same scenario is a no-op on the entity side.
- **maxRetries ceiling:** read from `multi-agent-eval.run.max-retries` (default 2). The workflow checks `retriesRemaining > 0` BEFORE scheduling `rerunStep`; it never recurses past the ceiling.
- **Rerun scope:** on a `FAIL` verdict, the workflow extracts the failing `scenarioId` list from `JudgmentNotes.bullets` and passes only those IDs to `RERUN_FAILING`, reducing orchestrator latency on subsequent attempts.
- **CI gate step:** `ciGateStep` is pure-function (no LLM call). It runs after every terminal transition (`passStep` or `failStep`) and emits `CIGateBlocked` when `finalScore < passThreshold` AND `ciGateBlocked` is not already set on the entity. The step is idempotent.
- **Saga semantics:** there are no external side-effects to compensate. The `failStep` preserves every scenario result and every judgment on the entity; an operator can inspect the full evidence without re-running.
