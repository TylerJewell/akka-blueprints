# PLAN — dev-team-task-board

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

  Lead[TeamLead]:::agent
  Dev[DeveloperAgent]:::agent

  Plan[PlanningWorkflow]:::wf
  DevWF[DeveloperWorkflow]:::wf

  Project[ProjectEntity]:::ese
  TaskE[TaskEntity]:::ese
  Mailbox[PeerMailbox]:::ese
  Intake[IntakeQueue]:::ese
  Control[SystemControl]:::kve
  Board[TaskBoardView]:::view
  Consumer[ProjectRequestConsumer]:::cons
  Sim[ProjectSimulator]:::ta
  Stuck[StuckTaskMonitor]:::ta
  API[DevTeamEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit project| Intake
  Sim -.->|every 60s| Intake
  Intake -.->|subscribes| Consumer
  Consumer -->|create + start| Project
  Consumer -->|start workflow| Plan
  Plan -->|call| Lead
  Plan -->|create one per task| TaskE
  TaskE -.->|projects| Board
  DevWF -->|poll board| Board
  DevWF -->|atomic claim| TaskE
  DevWF -->|call| Dev
  DevWF -->|post message| Mailbox
  DevWF -->|read flag| Control
  Stuck -.->|every 30s| TaskE
  API -->|halt / resume| Control
  API -->|reply| Mailbox
  API -->|query / SSE| Board
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions and scheduled ticks. `DeveloperAgent` is one agent class run as several instances (`dev-1`, `dev-2`, `dev-3`); each instance is driven by its own `DeveloperWorkflow`.

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as DevTeamEndpoint
  participant Q as IntakeQueue
  participant C as ProjectRequestConsumer
  participant PW as PlanningWorkflow
  participant L as TeamLead
  participant T as TaskEntity
  participant DW as DeveloperWorkflow
  participant D as DeveloperAgent
  participant V as TaskBoardView

  U->>API: POST /api/projects {title, description}
  API->>Q: submitProject(brief)
  API-->>U: 202 {projectId}
  Q->>C: ProjectSubmitted
  C->>PW: start({projectId})
  PW->>L: decompose(brief)
  L-->>PW: TaskPlan{tasks}
  PW->>T: createTask (one per spec, status OPEN)
  T-->>V: project rows
  Note over DW: developer loops already polling
  DW->>V: getAllTasks
  V-->>DW: OPEN task with deps DONE
  DW->>T: claim(dev-1)
  T-->>DW: TaskClaimed (won the race)
  DW->>D: implement(task)
  Note over D: every tool call passes the before-tool-call guardrail
  D-->>DW: CodeArtifact{files, summary}
  DW->>DW: testGate(artifact)
  DW->>T: passTest -> DONE
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
  IN_PROGRESS --> BLOCKED: tool guardrail refusal
  IN_PROGRESS --> OPEN: release (stuck > 2 min)
  IN_REVIEW --> DONE: testPassed
  IN_REVIEW --> IN_PROGRESS: testFailed (retry)
  IN_REVIEW --> BLOCKED: tests failed (retries exhausted)
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
  TaskEntity ||--o{ TestPassed : emits
  TaskEntity ||--o{ TestFailed : emits
  TaskEntity ||--o{ TaskBlocked : emits
  TaskEntity ||--o{ TaskReleased : emits
  TaskEntity ||--o{ TaskCompleted : emits
  TaskBoardView }o--|| TaskEntity : projects
  ProjectEntity ||--o{ ProjectCreated : emits
  ProjectEntity ||--o{ ProjectPlanned : emits
  ProjectEntity ||--o{ ProjectCompleted : emits
  ProjectEntity ||--o{ TaskEntity : "owns N tasks"
  IntakeQueue ||--o{ ProjectSubmitted : emits
  ProjectRequestConsumer }o--|| IntakeQueue : subscribes
  PeerMailbox ||--o{ MessagePosted : emits
  PeerMailbox ||--o{ MessageReplied : emits
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `TeamLead` | `application/TeamLead.java` |
| `DeveloperAgent` | `application/DeveloperAgent.java` |
| `TeamTasks` | `application/TeamTasks.java` |
| `TestRunner` | `application/TestRunner.java` |
| `PlanningWorkflow` | `application/PlanningWorkflow.java` |
| `DeveloperWorkflow` | `application/DeveloperWorkflow.java` |
| `TaskEntity` | `application/TaskEntity.java` (state in `domain/Task.java`, events in `domain/TaskEvent.java`) |
| `ProjectEntity` | `application/ProjectEntity.java` (state in `domain/Project.java`, events in `domain/ProjectEvent.java`) |
| `PeerMailbox` | `application/PeerMailbox.java` (state + events in `domain/`) |
| `IntakeQueue` | `application/IntakeQueue.java` |
| `SystemControl` | `application/SystemControl.java` |
| `TaskBoardView` | `application/TaskBoardView.java` |
| `ProjectRequestConsumer` | `application/ProjectRequestConsumer.java` |
| `ProjectSimulator` | `application/ProjectSimulator.java` |
| `StuckTaskMonitor` | `application/StuckTaskMonitor.java` |
| `DevTeamEndpoint` | `api/DevTeamEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `Bootstrap` | `Bootstrap.java` |

Akka component count: **2 autonomous-agent · 2 workflow · 4 event-sourced-entity · 1 key-value-entity · 1 view · 1 consumer · 2 timed-action · 2 http-endpoint · 1 service-setup**.

## Concurrency notes

- **Atomic claim is the whole pattern.** `TaskEntity` is a single-writer; `claim(devId)` emits `TaskClaimed` only when the current status is `OPEN`. Two developer workflows that read the same `OPEN` task from the view and both call `claim` are serialised by the entity — the first wins, the second receives the already-claimed `Task` and returns to polling. No lock, no external queue.
- **Workflow step timeouts:** `PlanningWorkflow.decomposeStep` and `DeveloperWorkflow.workStep` call agents, so each sets an explicit `stepTimeout` of 90 s (Lesson 4). The default 5 s timeout would expire mid-LLM-call.
- **Idle polling:** `DeveloperWorkflow.pollStep` self-schedules a 5 s resume timer when the team is halted or no eligible `OPEN` task exists, so an idle developer is a paused workflow, not a busy loop.
- **Dependency gate:** a task is eligible only when every title in its `dependsOn` resolves to a `DONE` task on the board. The poll filters on this client-side (the view exposes no enum-status filter — Lesson 2).
- **Release for liveness:** `StuckTaskMonitor` returns a task claimed-but-idle for more than two minutes to `OPEN`, so a developer that fails mid-task does not strand work. `release` is a no-op unless the task is `CLAIMED` or `IN_PROGRESS`.
- **Test gate:** `TestRunner` is a deterministic pure function (no LLM call) so the gate is reproducible; the same artifact always yields the same `TestReport`. The Maven `test` phase enforces the same gate at build time (control A1).
- **Halt:** `SystemControl` is read at the top of `pollStep` and inside the before-tool-call guardrail, so a halt both stops new claims and refuses any in-flight tool call (control HT1).
- **Idempotency:** deterministic `taskId = projectId + "-t" + index` makes `createTask` idempotent if `PlanningWorkflow.createTasksStep` is retried.
