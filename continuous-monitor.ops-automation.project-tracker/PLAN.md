# PLAN — project-tracker

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab.

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
  classDef gc fill:#1a0e0e,stroke:#ef4444,color:#ef4444;

  Poller[PlannerPoller]:::ta
  Queue[BoardEventQueue]:::ese
  Sync[BoardSyncConsumer]:::cons
  Assign[AssignmentAgent]:::agent
  Nudge[NudgeAgent]:::agent
  Judge[EvalJudge]:::agent
  WF[PlannerWorkflow]:::wf
  GC[GuardrailChecker]:::gc
  Entity[PlannerTaskEntity]:::ese
  View[BoardView]:::view
  Stale[StaleTaskChecker]:::ta
  Eval[EvalRunner]:::ta
  API[BoardEndpoint]:::ep
  App[AppEndpoint]:::ep

  Poller -.->|every 20s| Queue
  Queue -.->|subscribes| Sync
  Sync -->|emit TaskCreated| Entity
  Sync -->|start workflow| WF
  WF -->|call| Assign
  WF -->|check| GC
  WF -->|emit events| Entity
  Entity -.->|projects| View
  Stale -.->|every 5m| View
  Stale -->|call| Nudge
  Stale -->|check| GC
  Stale -->|emit NudgeSent/Escalated| Entity
  Eval -.->|every 60m| View
  Eval -->|call| Judge
  Eval -->|emit AssignmentEvalScored| Entity
  API -->|complete/escalate| Entity
  API -->|query/SSE| View
```

## Interaction sequence — J1 + J2

```mermaid
sequenceDiagram
  autonumber
  participant P as PlannerPoller
  participant Q as BoardEventQueue
  participant S as BoardSyncConsumer
  participant E as PlannerTaskEntity
  participant W as PlannerWorkflow
  participant A as AssignmentAgent
  participant G as GuardrailChecker
  participant SC as StaleTaskChecker
  participant N as NudgeAgent

  P->>Q: emit BoardEventReceived
  Q->>S: BoardEventReceived
  S->>E: emit TaskCreated
  S->>W: start({taskId, summary})
  W->>A: recommend(summary)
  A-->>W: AssignmentRecommendation
  W->>G: checkAssignment(rec, summary)
  G-->>W: DispatchResult(dispatched=true)
  W->>E: emit TaskAssigned
  Note over E,W: Workflow monitors for completion
  SC->>E: markOverdue (due date passed)
  SC->>N: draft nudge(task)
  N-->>SC: NudgeDraft
  SC->>G: checkNudge(draft, task)
  G-->>SC: DispatchResult(dispatched=true)
  SC->>E: emit NudgeSent
```

## State machine — `PlannerTaskEntity`

```mermaid
stateDiagram-v2
  [*] --> CREATED
  CREATED --> ASSIGNED: TaskAssigned
  CREATED --> CREATED: TaskAssignmentBlocked
  ASSIGNED --> IN_PROGRESS: TaskProgressStarted
  ASSIGNED --> OVERDUE: TaskOverdue
  IN_PROGRESS --> OVERDUE: TaskOverdue
  IN_PROGRESS --> COMPLETED: TaskCompleted
  OVERDUE --> NUDGED: NudgeSent
  NUDGED --> NUDGED: NudgeSent (nudgeCount < 3)
  NUDGED --> ESCALATED: NudgeSent (nudgeCount >= 3)
  NUDGED --> COMPLETED: TaskCompleted
  ESCALATED --> COMPLETED: TaskCompleted
  ESCALATED --> [*]
  COMPLETED --> [*]
  CANCELLED --> [*]
```

## Entity model

```mermaid
erDiagram
  PlannerTaskEntity ||--o{ TaskCreated : emits
  PlannerTaskEntity ||--o{ TaskAssigned : emits
  PlannerTaskEntity ||--o{ TaskAssignmentBlocked : emits
  PlannerTaskEntity ||--o{ TaskProgressStarted : emits
  PlannerTaskEntity ||--o{ TaskOverdue : emits
  PlannerTaskEntity ||--o{ NudgeSent : emits
  PlannerTaskEntity ||--o{ TaskEscalated : emits
  PlannerTaskEntity ||--o{ TaskCompleted : emits
  PlannerTaskEntity ||--o{ TaskCancelled : emits
  PlannerTaskEntity ||--o{ AssignmentEvalScored : emits
  BoardView }o--|| PlannerTaskEntity : projects
  BoardEventQueue ||--o{ BoardEventReceived : emits
  BoardSyncConsumer }o--|| BoardEventQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerPoller` | `application/PlannerPoller.java` |
| `BoardEventQueue` | `application/BoardEventQueue.java` |
| `BoardSyncConsumer` | `application/BoardSyncConsumer.java` |
| `AssignmentAgent` | `application/AssignmentAgent.java` |
| `NudgeAgent` | `application/NudgeAgent.java` |
| `EvalJudge` | `application/EvalJudge.java` |
| `PlannerWorkflow` | `application/PlannerWorkflow.java` |
| `GuardrailChecker` | `application/GuardrailChecker.java` |
| `PlannerTaskEntity` | `application/PlannerTaskEntity.java` (state in `domain/PlannerTask.java`, events in `domain/PlannerTaskEvent.java`) |
| `BoardView` | `application/BoardView.java` |
| `StaleTaskChecker` | `application/StaleTaskChecker.java` |
| `EvalRunner` | `application/EvalRunner.java` |
| `BoardEndpoint` | `api/BoardEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: recommend 15 s, guardrail check 5 s, assign 10 s. On timeout the workflow emits `TaskAssignmentBlocked` and ends.
- **Stale task scan**: `StaleTaskChecker` processes at most 3 nudge candidates per tick to avoid bursty LLM calls.
- **Idempotency**: every workflow uses `taskId` as the workflow id so duplicate `TaskCreated` events fold into one workflow.
- **Eval sampling**: per tick, EvalRunner picks up to 5 COMPLETED tasks with no `evalScore`, oldest-first.
- **Guardrail placement**: `GuardrailChecker` is a synchronous plain Java class called on the workflow thread and the StaleTaskChecker thread — no Akka component boundary.
