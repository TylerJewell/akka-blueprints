# PLAN — field-service-dispatcher

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

  Disp[DispatcherAgent]:::agent
  Route[RouteOptimizerAgent]:::agent
  Avail[AvailabilityAgent]:::agent

  WF[ScheduleWorkflow]:::wf
  Sched[ScheduleEntity]:::ese
  Ctrl[OperatorControlEntity]:::ese
  Queue[WorkOrderQueue]:::ese
  View[ScheduleView]:::view
  Consumer[WorkOrderConsumer]:::cons
  Sim[WorkOrderSimulator]:::ta
  Stalled[StalledScheduleMonitor]:::ta
  Fairness[FairnessMonitor]:::ta
  API[ScheduleEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|pause/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN_SCHEDULE / DECIDE_ASSIGNMENT / COMPOSE_SUMMARY| Disp
  WF -->|FIND_TECHNICIAN| Route
  WF -->|CHECK_AVAILABILITY| Avail
  WF -->|emit events| Sched
  WF -->|poll| Ctrl
  Sched -.->|projects| View
  API -->|query/SSE| View
  Stalled -.->|every 30s| Sched
  Fairness -.->|every 5m| Sched
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ScheduleEndpoint
  participant Q as WorkOrderQueue
  participant C as WorkOrderConsumer
  participant W as ScheduleWorkflow
  participant D as DispatcherAgent
  participant S as Specialist (Route/Availability)
  participant E as ScheduleEntity
  participant CTL as OperatorControlEntity
  participant V as ScheduleView

  U->>API: POST /api/schedules {description, serviceAddress, requiredSkill}
  API->>Q: append WorkOrderSubmitted
  API-->>U: 202 {scheduleId}
  Q->>C: WorkOrderSubmitted
  C->>W: start({scheduleId, description})
  W->>E: emit ScheduleCreated (PLANNING)
  W->>D: PLAN_SCHEDULE(description)
  D-->>W: ScheduleLedger
  W->>E: emit SchedulePlanned, status DISPATCHING
  loop until Complete | Fail | Pause
    W->>CTL: get pause flag
    CTL-->>W: paused=false
    W->>D: DECIDE_ASSIGNMENT(ledgers)
    D-->>W: Continue(AssignmentDecision)
    W->>W: guardrail.vet(decision)
    W->>S: runSingleTask(assignment)
    S-->>W: AssignmentResult
    W->>W: CredentialScrubber.scrub(content)
    W->>E: emit AssignmentRecorded (AssignmentEntry)
    W->>W: fairnessCheckStep inline check
  end
  W->>D: COMPOSE_SUMMARY
  D-->>W: DispatchSummary
  W->>E: emit ScheduleCompleted
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `ScheduleEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> DISPATCHING: SchedulePlanned
  DISPATCHING --> DISPATCHING: AssignmentRecorded / AssignmentBlocked / LedgerRevised / FairnessAlertRecorded
  DISPATCHING --> COMPLETED: ScheduleCompleted
  DISPATCHING --> FAILED: ScheduleFailed
  DISPATCHING --> PAUSED: SchedulePausedOperator
  DISPATCHING --> STALLED: ScheduleFailedTimeout
  STALLED --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  PAUSED --> [*]
```

## Entity model

```mermaid
erDiagram
  ScheduleEntity ||--o{ SchedulePlanned : emits
  ScheduleEntity ||--o{ AssignmentDispatched : emits
  ScheduleEntity ||--o{ AssignmentBlocked : emits
  ScheduleEntity ||--o{ AssignmentRecorded : emits
  ScheduleEntity ||--o{ LedgerRevised : emits
  ScheduleEntity ||--o{ FairnessAlertRecorded : emits
  ScheduleEntity ||--o{ ScheduleCompleted : emits
  ScheduleEntity ||--o{ ScheduleFailed : emits
  ScheduleEntity ||--o{ SchedulePausedOperator : emits
  ScheduleEntity ||--o{ ScheduleFailedTimeout : emits
  ScheduleView }o--|| ScheduleEntity : projects
  OperatorControlEntity ||--o{ PauseRequested : emits
  OperatorControlEntity ||--o{ PauseCleared : emits
  WorkOrderQueue ||--o{ WorkOrderSubmitted : emits
  WorkOrderConsumer }o--|| WorkOrderQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `DispatcherAgent` | `application/DispatcherAgent.java` |
| `RouteOptimizerAgent` | `application/RouteOptimizerAgent.java` |
| `AvailabilityAgent` | `application/AvailabilityAgent.java` |
| `ScheduleWorkflow` | `application/ScheduleWorkflow.java` |
| `ScheduleEntity` | `application/ScheduleEntity.java` (state in `domain/Schedule.java`, events in `domain/ScheduleEvent.java`) |
| `OperatorControlEntity` | `application/OperatorControlEntity.java` |
| `WorkOrderQueue` | `application/WorkOrderQueue.java` |
| `ScheduleView` | `application/ScheduleView.java` |
| `WorkOrderConsumer` | `application/WorkOrderConsumer.java` |
| `WorkOrderSimulator` | `application/WorkOrderSimulator.java` |
| `StalledScheduleMonitor` | `application/StalledScheduleMonitor.java` |
| `FairnessMonitor` | `application/FairnessMonitor.java` |
| `AssignmentGuardrail` | `application/AssignmentGuardrail.java` |
| `CredentialScrubber` | `application/CredentialScrubber.java` |
| `DispatcherTasks` | `application/DispatcherTasks.java` |
| `SpecialistTasks` | `application/SpecialistTasks.java` |
| `ScheduleEndpoint` | `api/ScheduleEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s (covers any specialist call), `decideStep` 45 s, `completeStep` 60 s. Default recovery: `maxRetries(2).failoverTo(ScheduleWorkflow::error)`.
- **Replan budget:** the dispatcher may emit `Replan` at most twice in a row without a `Continue` in between; a third consecutive `Replan` is treated as `Fail`.
- **Failure budget:** the dispatcher may emit `Continue` on the same `(specialist, assignment)` at most three times; a fourth attempt is treated as `Fail`.
- **Pause poll:** every `checkPauseStep` reads `OperatorControlEntity.get` synchronously — no caching. An operator pause arriving during a `dispatchStep` lets the in-flight step finish; the loop exits at the next `checkPauseStep`.
- **Idempotency:** `ScheduleEndpoint.submit` uses `(description, serviceAddress, requiredSkill)` over a 10 s window to deduplicate `POST /api/schedules`.
- **Stalled detection:** `StalledScheduleMonitor` ticks every 30 s; `ScheduleFailedTimeout` is non-fatal to other schedules.
- **Fairness monitor:** runs every 5 minutes; non-blocking to in-progress schedules; alerts are advisory records on the entity.
- **Scrubber determinism:** `CredentialScrubber.scrub` is pure; it never inspects external state. The same input always yields the same scrubbed output, keeping `AssignmentEntry` events deterministic and replayable.
