# PLAN — code-assistant-loop

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

  Plan[PlannerAgent]:::agent
  Read[ReaderAgent]:::agent
  Edit[EditorAgent]:::agent
  Run[RunnerAgent]:::agent

  WF[SessionWorkflow]:::wf
  Sess[SessionEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[TaskQueue]:::ese
  View[SessionView]:::view
  Consumer[TaskRequestConsumer]:::cons
  Sim[TaskSimulator]:::ta
  Stuck[StuckSessionMonitor]:::ta
  API[SessionEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|DRAFT_PLAN / DECIDE_ACTION / COMPOSE_COMMIT| Plan
  WF -->|READ_FILE| Read
  WF -->|EDIT_FILE| Edit
  WF -->|RUN_TESTS| Run
  WF -->|emit events| Sess
  WF -->|poll| Ctrl
  Sess -.->|projects| View
  API -->|query/SSE| View
  Stuck -.->|every 30s| Sess
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as SessionEndpoint
  participant Q as TaskQueue
  participant C as TaskRequestConsumer
  participant W as SessionWorkflow
  participant P as PlannerAgent
  participant A as Agent (Reader/Editor/Runner)
  participant E as SessionEntity
  participant CTL as SystemControlEntity
  participant V as SessionView

  U->>API: POST /api/sessions {prompt}
  API->>Q: append TaskSubmitted
  API-->>U: 202 {sessionId}
  Q->>C: TaskSubmitted
  C->>W: start({sessionId, prompt})
  W->>E: emit SessionCreated (PLANNING)
  W->>P: DRAFT_PLAN(prompt)
  P-->>W: PlanLedger
  W->>E: emit SessionPlanned, status EXECUTING
  loop until Commit | Fail | Halt
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>P: DECIDE_ACTION(ledgers)
    P-->>W: Continue(ActionDecision)
    W->>W: guardrail.vet(decision)
    W->>A: runSingleTask(action)
    A-->>W: ActionResult
    W->>E: emit EditRecorded (EditEntry)
    W->>W: CiGate.evaluate(entry) if RUN_TESTS
    W->>W: autoHaltEval(counters)
  end
  W->>P: COMPOSE_COMMIT
  P-->>W: CommitSummary
  W->>E: emit SessionCommitted
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> EXECUTING: SessionPlanned
  EXECUTING --> EXECUTING: EditRecorded / EditBlocked / CiGateFailed / PlanRevised
  EXECUTING --> COMMITTED: SessionCommitted
  EXECUTING --> FAILED: SessionFailed
  EXECUTING --> HALTED: SessionHaltedOperator / SessionHaltedAutomatic
  EXECUTING --> STUCK: SessionFailedTimeout
  STUCK --> [*]
  COMMITTED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionPlanned : emits
  SessionEntity ||--o{ ActionDispatched : emits
  SessionEntity ||--o{ EditBlocked : emits
  SessionEntity ||--o{ CiGateFailed : emits
  SessionEntity ||--o{ EditRecorded : emits
  SessionEntity ||--o{ PlanRevised : emits
  SessionEntity ||--o{ SessionCommitted : emits
  SessionEntity ||--o{ SessionFailed : emits
  SessionEntity ||--o{ SessionHaltedAutomatic : emits
  SessionEntity ||--o{ SessionHaltedOperator : emits
  SessionEntity ||--o{ SessionFailedTimeout : emits
  SessionView }o--|| SessionEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  TaskQueue ||--o{ TaskSubmitted : emits
  TaskRequestConsumer }o--|| TaskQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `ReaderAgent` | `application/ReaderAgent.java` |
| `EditorAgent` | `application/EditorAgent.java` |
| `RunnerAgent` | `application/RunnerAgent.java` |
| `SessionWorkflow` | `application/SessionWorkflow.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `TaskQueue` | `application/TaskQueue.java` |
| `SessionView` | `application/SessionView.java` |
| `TaskRequestConsumer` | `application/TaskRequestConsumer.java` |
| `TaskSimulator` | `application/TaskSimulator.java` |
| `StuckSessionMonitor` | `application/StuckSessionMonitor.java` |
| `EditGuardrail` | `application/EditGuardrail.java` |
| `CiGate` | `application/CiGate.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `AgentTasks` | `application/AgentTasks.java` |
| `SessionEndpoint` | `api/SessionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s (covers any agent call), `decideStep` 45 s, `commitStep` 60 s. Default recovery: `maxRetries(2).failoverTo(SessionWorkflow::error)`.
- **Replan budget:** the planner may emit `Replan` at most twice in a row without a successful edit in between; a third consecutive `Replan` is treated as `Fail`.
- **CI failure budget:** three consecutive `CiGateFailed` entries without an intervening `OK` edit causes the workflow to transition to `failStep`.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. A halt arriving during a `dispatchStep` lets the in-flight action finish; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `SessionEndpoint.submit` deduplicates `POST /api/sessions` on `(prompt, requestedBy)` over a 10 s window.
- **Stuck detection:** `StuckSessionMonitor` ticks every 30 s; tasks `EXECUTING` for > 5 minutes are marked `STUCK`.
- **CiGate determinism:** `CiGate.evaluate` is pure; it never calls the LLM. The same test-output string always yields the same `CiVerdict`, keeping `EditEntry` events deterministic and replayable.
