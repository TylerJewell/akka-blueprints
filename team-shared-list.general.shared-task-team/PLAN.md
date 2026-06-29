# PLAN — shared-task-team

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab with the Akka theme variables and the Lesson 24 state-label CSS overrides.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef kve fill:#1f1900,stroke:#C9A227,color:#C9A227;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Planner[GoalPlanner]:::agent
  Worker[WorkerAgent]:::agent

  Plan[PlanningWorkflow]:::wf
  WorkWF[WorkerWorkflow]:::wf

  GoalE[GoalEntity]:::ese
  TaskE[TaskEntity]:::ese
  Mailbox[WorkerMailbox]:::ese
  Intake[IntakeQueue]:::ese
  Control[SystemControl]:::kve
  Board[TaskBoardView]:::view
  Consumer[GoalRequestConsumer]:::cons
  Sim[GoalSimulator]:::ta
  Stuck[StuckTaskMonitor]:::ta
  API[SharedTaskEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit goal| Intake
  Sim -.->|every 60s| Intake
  Intake -.->|subscribes| Consumer
  Consumer -->|create + start| GoalE
  Consumer -->|start workflow| Plan
  Plan -->|call| Planner
  Plan -->|create one per task| TaskE
  TaskE -.->|projects| Board
  WorkWF -->|poll board| Board
  WorkWF -->|atomic claim| TaskE
  WorkWF -->|call| Worker
  WorkWF -->|post message| Mailbox
  WorkWF -->|read flag| Control
  Stuck -.->|every 30s| TaskE
  API -->|halt / resume| Control
  API -->|reply| Mailbox
  API -->|query / SSE| Board
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions and scheduled ticks. `WorkerAgent` is one agent class run as several instances (`worker-1`, `worker-2`, `worker-3`); each instance is driven by its own `WorkerWorkflow`.

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as SharedTaskEndpoint
  participant Q as IntakeQueue
  participant C as GoalRequestConsumer
  participant PW as PlanningWorkflow
  participant GP as GoalPlanner
  participant T as TaskEntity
  participant WW as WorkerWorkflow
  participant WA as WorkerAgent
  participant V as TaskBoardView

  U->>API: POST /api/goals {title, description}
  API->>Q: submitGoal(brief)
  API-->>U: 202 {goalId}
  Q->>C: GoalSubmitted
  C->>PW: start({goalId})
  PW->>GP: plan(brief)
  GP-->>PW: TaskPlan{tasks}
  PW->>T: createTask (one per spec, status OPEN)
  T-->>V: task rows
  Note over WW: worker loops already polling
  WW->>V: getAllTasks
  V-->>WW: OPEN task with deps DONE
  WW->>T: claim(worker-1)
  T-->>WW: TaskClaimed (won the race)
  WW->>WA: work(task)
  Note over WA: response passes before-agent-response guardrail
  WA-->>WW: ResultArtifact{sections, summary}
  WW->>WW: qualityCheck(artifact)
  WW->>T: passQuality -> DONE
  T-->>V: row DONE
  V-->>U: SSE update
```

## State machine — `TaskEntity`

```mermaid
stateDiagram-v2
  [*] --> OPEN
  OPEN --> CLAIMED: claim (atomic, single winner)
  CLAIMED --> IN_PROGRESS: start
  CLAIMED --> OPEN: release (stuck > 2 min)
  IN_PROGRESS --> IN_REVIEW: submitArtifact
  IN_PROGRESS --> BLOCKED: peer request raised
  IN_PROGRESS --> BLOCKED: response guardrail refusal
  IN_PROGRESS --> OPEN: release (stuck > 2 min)
  IN_REVIEW --> DONE: qualityPassed
  IN_REVIEW --> IN_PROGRESS: qualityFailed (retry)
  IN_REVIEW --> BLOCKED: quality checks failed (retries exhausted)
  BLOCKED --> OPEN: peer reply unblocks / operator reopen
  DONE --> [*]
```

## Entity model

```mermaid
erDiagram
  TaskEntity ||--o{ TaskCreated : emits
  TaskEntity ||--o{ TaskClaimed : emits
  TaskEntity ||--o{ TaskStarted : emits
  TaskEntity ||--o{ ArtifactSubmitted : emits
  TaskEntity ||--o{ QualityPassed : emits
  TaskEntity ||--o{ QualityFailed : emits
  TaskEntity ||--o{ TaskBlocked : emits
  TaskEntity ||--o{ TaskReleased : emits
  TaskEntity ||--o{ TaskCompleted : emits
  TaskBoardView }o--|| TaskEntity : projects
  GoalEntity ||--o{ GoalCreated : emits
  GoalEntity ||--o{ GoalPlanned : emits
  GoalEntity ||--o{ GoalCompleted : emits
  GoalEntity ||--o{ TaskEntity : "owns N tasks"
  IntakeQueue ||--o{ GoalSubmitted : emits
  GoalRequestConsumer }o--|| IntakeQueue : subscribes
  WorkerMailbox ||--o{ MessagePosted : emits
  WorkerMailbox ||--o{ MessageReplied : emits
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `GoalPlanner` | `application/GoalPlanner.java` |
| `WorkerAgent` | `application/WorkerAgent.java` |
| `TeamTasks` | `application/TeamTasks.java` |
| `QualityChecker` | `application/QualityChecker.java` |
| `PlanningWorkflow` | `application/PlanningWorkflow.java` |
| `WorkerWorkflow` | `application/WorkerWorkflow.java` |
| `TaskEntity` | `application/TaskEntity.java` (state in `domain/Task.java`, events in `domain/TaskEvent.java`) |
| `GoalEntity` | `application/GoalEntity.java` (state in `domain/Goal.java`, events in `domain/GoalEvent.java`) |
| `WorkerMailbox` | `application/WorkerMailbox.java` (state + events in `domain/`) |
| `IntakeQueue` | `application/IntakeQueue.java` |
| `SystemControl` | `application/SystemControl.java` |
| `TaskBoardView` | `application/TaskBoardView.java` |
| `GoalRequestConsumer` | `application/GoalRequestConsumer.java` |
| `GoalSimulator` | `application/GoalSimulator.java` |
| `StuckTaskMonitor` | `application/StuckTaskMonitor.java` |
| `SharedTaskEndpoint` | `api/SharedTaskEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `Bootstrap` | `Bootstrap.java` |

Akka component count: **2 autonomous-agent · 2 workflow · 4 event-sourced-entity · 1 key-value-entity · 1 view · 1 consumer · 2 timed-action · 2 http-endpoint · 1 service-setup**.

## Concurrency notes

- **Atomic claim is the whole pattern.** `TaskEntity` is a single-writer; `claim(workerId)` emits `TaskClaimed` only when the current status is `OPEN`. Two worker workflows that read the same `OPEN` task from the view and both call `claim` are serialised by the entity — the first wins, the second receives the already-claimed `Task` and returns to polling. No lock, no external queue.
- **Workflow step timeouts:** `PlanningWorkflow.decomposeStep` and `WorkerWorkflow.workStep` call agents, so each sets an explicit `stepTimeout` of 90 s (Lesson 4). The default 5 s timeout would expire mid-LLM-call.
- **Idle polling:** `WorkerWorkflow.pollStep` self-schedules a 5 s resume timer when the team is halted or no eligible `OPEN` task exists, so an idle worker is a paused workflow, not a busy loop.
- **Dependency gate:** a task is eligible only when every title in its `dependsOn` resolves to a `DONE` task on the board. The poll filters on this client-side (the view exposes no enum-status filter — Lesson 2).
- **Release for liveness:** `StuckTaskMonitor` returns a task claimed-but-idle for more than two minutes to `OPEN`, so a worker that fails mid-task does not strand work. `release` is a no-op unless the task is `CLAIMED` or `IN_PROGRESS`.
- **Quality check:** `QualityChecker` is a deterministic pure function (no LLM call) so the gate is reproducible; the same artifact always yields the same `QualityReport`. The Maven `test` phase enforces the same gate at build time.
- **Halt:** `SystemControl` is read at the top of `pollStep` and inside the before-agent-response guardrail, so a halt both stops new claims and refuses any in-flight response generation (control HT1).
- **Idempotency:** deterministic `taskId = goalId + "-t" + index` makes `createTask` idempotent if `PlanningWorkflow.createTasksStep` is retried.
- **On-decision eval:** after a goal completes, the eval scores consistency across worker contributions. It is non-blocking — it records results alongside `GoalEntity` but does not gate the `COMPLETED` transition.
