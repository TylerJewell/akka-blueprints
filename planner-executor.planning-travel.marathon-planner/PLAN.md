# PLAN — marathon-planner

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

  Planner[TrainingPlannerAgent]:::agent
  Builder[WorkoutBuilderAgent]:::agent
  Assessor[FitnessAssessorAgent]:::agent

  WF[PlanWorkflow]:::wf
  Runner[RunnerEntity]:::ese
  Registry[SkillRegistryEntity]:::ese
  Queue[PlanQueue]:::ese
  View[PlanView]:::view
  Consumer[PlanRequestConsumer]:::cons
  Sim[PlanSimulator]:::ta
  Stale[StaleWorkoutMonitor]:::ta
  API[PlanEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|load/unload skill| Registry
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|DRAFT_PLAN / DECIDE_NEXT / COMPOSE_SUMMARY| Planner
  WF -->|BUILD_WORKOUTS| Builder
  WF -->|ASSESS| Assessor
  WF -->|emit events| Runner
  WF -->|poll registry| Registry
  Runner -.->|projects| View
  API -->|query/SSE| View
  Stale -.->|every 30s| Runner
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as PlanEndpoint
  participant Q as PlanQueue
  participant C as PlanRequestConsumer
  participant W as PlanWorkflow
  participant A as FitnessAssessorAgent
  participant P as TrainingPlannerAgent
  participant B as WorkoutBuilderAgent
  participant E as RunnerEntity
  participant R as SkillRegistryEntity
  participant V as PlanView

  U->>API: POST /api/plans {profile}
  API->>Q: append PlanRequested
  API-->>U: 202 {planId}
  Q->>C: PlanRequested
  C->>W: start({planId, profile})
  W->>E: emit RunnerPlanCreated (ASSESSING)
  W->>A: ASSESS(profile)
  A-->>W: FitnessBaseline
  W->>E: emit BaselineAssessed, status PLANNING
  W->>P: DRAFT_PLAN(profile, baseline)
  P-->>W: PlanLedger
  W->>E: emit PlanDrafted, status BUILDING
  loop until Complete | Fail | Stale
    W->>P: DECIDE_NEXT(ledgers)
    P-->>W: Continue(SkillDispatch)
    W->>R: getRegistry
    R-->>W: SkillRegistry
    W->>W: CapabilityGuardrail.vet(dispatch, registry)
    W->>B: BUILD_WORKOUTS(phaseStub)
    B-->>W: WorkoutResult
    W->>W: AdherenceEvaluator.score(result, ledger)
    W->>E: emit AdjustmentRecorded (AdjustmentEntry)
  end
  W->>P: COMPOSE_SUMMARY
  P-->>W: PlanSummary
  W->>E: emit PlanCompleted
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `RunnerEntity`

```mermaid
stateDiagram-v2
  [*] --> ASSESSING
  ASSESSING --> PLANNING: BaselineAssessed
  PLANNING --> BUILDING: PlanDrafted
  BUILDING --> BUILDING: WorkoutBuilt / WorkoutBlocked / AdjustmentRecorded / LedgerRevised
  BUILDING --> COMPLETED: PlanCompleted
  BUILDING --> FAILED: PlanFailed
  BUILDING --> STALE: PlanFailedTimeout
  STALE --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  RunnerEntity ||--o{ RunnerPlanCreated : emits
  RunnerEntity ||--o{ BaselineAssessed : emits
  RunnerEntity ||--o{ PlanDrafted : emits
  RunnerEntity ||--o{ WorkoutBuilt : emits
  RunnerEntity ||--o{ WorkoutBlocked : emits
  RunnerEntity ||--o{ LedgerRevised : emits
  RunnerEntity ||--o{ AdjustmentRecorded : emits
  RunnerEntity ||--o{ PlanCompleted : emits
  RunnerEntity ||--o{ PlanFailed : emits
  RunnerEntity ||--o{ PlanFailedTimeout : emits
  PlanView }o--|| RunnerEntity : projects
  SkillRegistryEntity ||--o{ SkillLoaded : emits
  SkillRegistryEntity ||--o{ SkillUnloaded : emits
  PlanQueue ||--o{ PlanRequested : emits
  PlanRequestConsumer }o--|| PlanQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `TrainingPlannerAgent` | `application/TrainingPlannerAgent.java` |
| `WorkoutBuilderAgent` | `application/WorkoutBuilderAgent.java` |
| `FitnessAssessorAgent` | `application/FitnessAssessorAgent.java` |
| `PlanWorkflow` | `application/PlanWorkflow.java` |
| `RunnerEntity` | `application/RunnerEntity.java` (state in `domain/RunnerPlan.java`, events in `domain/PlanEvent.java`) |
| `SkillRegistryEntity` | `application/SkillRegistryEntity.java` |
| `PlanQueue` | `application/PlanQueue.java` |
| `PlanView` | `application/PlanView.java` |
| `PlanRequestConsumer` | `application/PlanRequestConsumer.java` |
| `PlanSimulator` | `application/PlanSimulator.java` |
| `StaleWorkoutMonitor` | `application/StaleWorkoutMonitor.java` |
| `CapabilityGuardrail` | `application/CapabilityGuardrail.java` |
| `AdherenceEvaluator` | `application/AdherenceEvaluator.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `ExecutorTasks` | `application/ExecutorTasks.java` |
| `PlanEndpoint` | `api/PlanEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `assessStep` 60 s, `planStep` 60 s, `guardStep` 10 s, `dispatchStep` 120 s, `evalStep` 30 s, `decideStep` 45 s, `summariseStep` 60 s. Default recovery: `maxRetries(2).failoverTo(PlanWorkflow::error)`.
- **Replan budget:** the planner may emit `Replan` at most twice in a row without a `Continue` in between; a third consecutive `Replan` becomes `Fail`.
- **Failure budget:** the planner may dispatch the same `(skill, task)` at most three times; a fourth attempt becomes `Fail`.
- **Registry poll:** every `guardStep` reads `SkillRegistryEntity.getRegistry` synchronously — no caching. A skill removed between loop ticks is caught on the next iteration.
- **Adherence threshold:** configurable via `application.conf adherence.threshold = 0.6`. Values below threshold trigger `LowAdherenceSignal`; the planner sees it on its next `DECIDE_NEXT` call.
- **Stale detection:** `StaleWorkoutMonitor` ticks every 30 s; plans in `BUILDING` for > 5 minutes are marked `STALE`. The workflow's `decideStep` checks the entity's status and exits when it reads `STALE`.
- **Idempotency:** `PlanEndpoint.submit` uses `(runnerId, goalRace, goalRaceDate)` over a 10 s window to deduplicate `POST /api/plans`.
- **AdherenceEvaluator determinism:** `AdherenceEvaluator.score` is pure — the same inputs always yield the same score, keeping `AdjustmentEntry` events deterministic and replayable.
