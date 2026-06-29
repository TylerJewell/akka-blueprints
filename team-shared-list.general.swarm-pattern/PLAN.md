# PLAN — swarm-pattern

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

  Coord[Coordinator]:::agent
  Wrk[Worker]:::agent

  Decomp[DecompositionWorkflow]:::wf
  WrkWF[WorkerWorkflow]:::wf

  Job[JobEntity]:::ese
  Item[WorkItemEntity]:::ese
  Relay[RelayMailbox]:::ese
  Log[SubmissionLog]:::ese
  Ctrl[SwarmControl]:::kve
  Board[WorkListView]:::view
  Consumer[JobRequestConsumer]:::cons
  Sim[JobSimulator]:::ta
  Monitor[StalledItemMonitor]:::ta
  API[SwarmEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit job| Log
  Sim -.->|every 60s| Log
  Log -.->|subscribes| Consumer
  Consumer -->|create + start| Job
  Consumer -->|start workflow| Decomp
  Decomp -->|call| Coord
  Decomp -->|create one per item| Item
  Item -.->|projects| Board
  WrkWF -->|poll board| Board
  WrkWF -->|atomic claim| Item
  WrkWF -->|call| Wrk
  WrkWF -->|post relay| Relay
  WrkWF -->|read flag| Ctrl
  Monitor -.->|every 30s| Item
  API -->|pause / resume| Ctrl
  API -->|reply| Relay
  API -->|query / SSE| Board
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions and scheduled ticks. `Worker` is one agent class run as several instances (`worker-1`, `worker-2`, `worker-3`); each instance is driven by its own `WorkerWorkflow`.

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as SwarmEndpoint
  participant L as SubmissionLog
  participant C as JobRequestConsumer
  participant DW as DecompositionWorkflow
  participant CO as Coordinator
  participant I as WorkItemEntity
  participant WW as WorkerWorkflow
  participant W as Worker
  participant V as WorkListView

  U->>API: POST /api/jobs {title, description}
  API->>L: submitJob(brief)
  API-->>U: 202 {jobId}
  L->>C: JobSubmitted
  C->>DW: start({jobId})
  DW->>CO: decompose(brief)
  CO-->>DW: WorkPlan{items}
  DW->>I: createItem (one per spec, status OPEN)
  I-->>V: item rows
  Note over WW: worker loops already polling
  WW->>V: getAllItems
  V-->>WW: OPEN item with deps DONE
  WW->>I: claim(worker-1)
  I-->>WW: WorkItemClaimed (won the race)
  WW->>W: process(item)
  W-->>WW: WorkResult{outputs, summary}
  WW->>WW: qualityGate(result)
  WW->>I: passQuality -> DONE
  I-->>V: row DONE
  V-->>U: SSE update
```

## State machine — `WorkItemEntity`

```mermaid
stateDiagram-v2
  [*] --> OPEN
  OPEN --> CLAIMED: claim (atomic, single winner)
  CLAIMED --> IN_PROGRESS: start
  CLAIMED --> OPEN: release (idle > 2 min)
  IN_PROGRESS --> IN_REVIEW: submitResult
  IN_PROGRESS --> STALLED: relay request raised
  IN_PROGRESS --> OPEN: release (idle > 2 min)
  IN_REVIEW --> DONE: qualityPassed
  IN_REVIEW --> IN_PROGRESS: qualityFailed (retry)
  IN_REVIEW --> STALLED: quality failed (retries exhausted)
  STALLED --> OPEN: relay reply / operator reopen
  DONE --> [*]
```

## Entity model

```mermaid
erDiagram
  WorkItemEntity ||--o{ WorkItemCreated : emits
  WorkItemEntity ||--o{ WorkItemClaimed : emits
  WorkItemEntity ||--o{ WorkItemStarted : emits
  WorkItemEntity ||--o{ ResultSubmitted : emits
  WorkItemEntity ||--o{ QualityPassed : emits
  WorkItemEntity ||--o{ QualityFailed : emits
  WorkItemEntity ||--o{ WorkItemStalled : emits
  WorkItemEntity ||--o{ WorkItemReleased : emits
  WorkItemEntity ||--o{ WorkItemCompleted : emits
  WorkListView }o--|| WorkItemEntity : projects
  JobEntity ||--o{ JobCreated : emits
  JobEntity ||--o{ JobPlanned : emits
  JobEntity ||--o{ JobFinished : emits
  JobEntity ||--o{ WorkItemEntity : "owns N items"
  SubmissionLog ||--o{ JobSubmitted : emits
  JobRequestConsumer }o--|| SubmissionLog : subscribes
  RelayMailbox ||--o{ RelayPosted : emits
  RelayMailbox ||--o{ RelayReplied : emits
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `Coordinator` | `application/Coordinator.java` |
| `Worker` | `application/Worker.java` |
| `SwarmTasks` | `application/SwarmTasks.java` |
| `QualityChecker` | `application/QualityChecker.java` |
| `DecompositionWorkflow` | `application/DecompositionWorkflow.java` |
| `WorkerWorkflow` | `application/WorkerWorkflow.java` |
| `WorkItemEntity` | `application/WorkItemEntity.java` (state in `domain/WorkItem.java`, events in `domain/WorkItemEvent.java`) |
| `JobEntity` | `application/JobEntity.java` (state in `domain/Job.java`, events in `domain/JobEvent.java`) |
| `RelayMailbox` | `application/RelayMailbox.java` (state + events in `domain/`) |
| `SubmissionLog` | `application/SubmissionLog.java` |
| `SwarmControl` | `application/SwarmControl.java` |
| `WorkListView` | `application/WorkListView.java` |
| `JobRequestConsumer` | `application/JobRequestConsumer.java` |
| `JobSimulator` | `application/JobSimulator.java` |
| `StalledItemMonitor` | `application/StalledItemMonitor.java` |
| `SwarmEndpoint` | `api/SwarmEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `Bootstrap` | `Bootstrap.java` |

Akka component count: **2 autonomous-agent · 2 workflow · 4 event-sourced-entity · 1 key-value-entity · 1 view · 1 consumer · 2 timed-action · 2 http-endpoint · 1 service-setup**.

## Concurrency notes

- **Atomic claim is the whole pattern.** `WorkItemEntity` is a single-writer; `claim(workerId)` emits `WorkItemClaimed` only when the current status is `OPEN`. Two worker workflows that read the same `OPEN` item from the view and both call `claim` are serialised by the entity — the first wins, the second receives the already-claimed `WorkItem` and returns to polling. No lock, no external queue.
- **Workflow step timeouts:** `DecompositionWorkflow.decomposeStep` and `WorkerWorkflow.workStep` call agents, so each sets an explicit `stepTimeout` of 90 s (Lesson 4). The default 5 s timeout would expire mid-LLM-call.
- **Idle polling:** `WorkerWorkflow.pollStep` self-schedules a 5 s resume timer when the swarm is paused or no eligible `OPEN` item exists, so an idle worker is a paused workflow, not a busy loop.
- **Dependency gate:** an item is eligible only when every title in its `dependsOn` resolves to a `DONE` item on the board. The poll filters on this client-side (the view exposes no enum-status filter — Lesson 2).
- **Release for liveness:** `StalledItemMonitor` returns an item claimed-but-idle for more than two minutes to `OPEN`, so a worker that fails mid-item does not strand work. `release` is a no-op unless the item is `CLAIMED` or `IN_PROGRESS`.
- **Quality gate:** `QualityChecker` is a deterministic pure function (no LLM call) so the gate is reproducible; the same result always yields the same `QualityReport`.
- **Pause:** `SwarmControl` is read at the top of `pollStep`, so a pause both stops new claims and blocks the worker loop while paused.
- **Idempotency:** deterministic `itemId = jobId + "-i" + index` makes `createItem` idempotent if `DecompositionWorkflow.createItemsStep` is retried.
