# PLAN — sdlc-task-planner

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

  Planner[PlannerAgent]:::agent
  Analyst[AnalystAgent]:::agent
  Arch[ArchitectAgent]:::agent
  Coder[CoderAgent]:::agent
  Reviewer[ReviewerAgent]:::agent

  WF[PlanWorkflow]:::wf
  PlanE[PlanEntity]:::ese
  Ctrl[SystemControlEntity]:::ese
  Queue[RequestQueue]:::ese
  View[PlanView]:::view
  Consumer[PlanRequestConsumer]:::cons
  Sim[RequestSimulator]:::ta
  Stale[StalePlanMonitor]:::ta
  API[PlanEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|halt/resume| Ctrl
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|DECOMPOSE / DECIDE / COMPOSE| Planner
  WF -->|ANALYSE_REQUIREMENTS| Analyst
  WF -->|DESIGN_COMPONENT| Arch
  WF -->|WRITE_CODE| Coder
  WF -->|REVIEW_ARTEFACT| Reviewer
  WF -->|emit events| PlanE
  WF -->|poll| Ctrl
  PlanE -.->|projects| View
  API -->|query/SSE| View
  Stale -.->|every 30s| PlanE
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as PlanEndpoint
  participant Q as RequestQueue
  participant C as PlanRequestConsumer
  participant W as PlanWorkflow
  participant P as PlannerAgent
  participant S as Specialist (Analyst/Architect/Coder/Reviewer)
  participant E as PlanEntity
  participant CTL as SystemControlEntity
  participant V as PlanView

  U->>API: POST /api/plans {prompt}
  API->>Q: append PlanSubmitted
  API-->>U: 202 {planId}
  Q->>C: PlanSubmitted
  C->>W: start({planId, prompt})
  W->>E: emit PlanCreated (PLANNING)
  W->>P: DECOMPOSE(prompt)
  P-->>W: TaskLedger
  W->>E: emit PlanDecomposed, status EXECUTING
  loop until Complete | Fail | Halt
    W->>CTL: get halt flag
    CTL-->>W: halted=false
    W->>P: DECIDE(ledgers)
    P-->>W: Continue(DispatchDecision)
    W->>W: guardrail.vet(decision)
    W->>S: runSingleTask(subtask)
    S-->>W: SubtaskResult
    W->>W: SecretScrubber.scrub(content)
    W->>E: emit SubtaskRecorded (ProgressEntry)
    W->>W: SafetyEvaluator.evaluate(entry)
  end
  W->>P: COMPOSE_DELIVERABLE
  P-->>W: PlanDeliverable
  W->>E: emit PlanCompleted
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `PlanEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> EXECUTING: PlanDecomposed
  EXECUTING --> EXECUTING: SubtaskRecorded / SubtaskBlocked / LedgerRevised
  EXECUTING --> COMPLETED: PlanCompleted
  EXECUTING --> FAILED: PlanFailed
  EXECUTING --> HALTED: PlanHaltedOperator / PlanHaltedAutomatic
  EXECUTING --> STUCK: PlanFailedTimeout
  STUCK --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  PlanEntity ||--o{ PlanCreated : emits
  PlanEntity ||--o{ PlanDecomposed : emits
  PlanEntity ||--o{ SubtaskDispatched : emits
  PlanEntity ||--o{ SubtaskBlocked : emits
  PlanEntity ||--o{ SubtaskRecorded : emits
  PlanEntity ||--o{ LedgerRevised : emits
  PlanEntity ||--o{ PlanCompleted : emits
  PlanEntity ||--o{ PlanFailed : emits
  PlanEntity ||--o{ PlanHaltedAutomatic : emits
  PlanEntity ||--o{ PlanHaltedOperator : emits
  PlanEntity ||--o{ PlanFailedTimeout : emits
  PlanView }o--|| PlanEntity : projects
  SystemControlEntity ||--o{ HaltRequested : emits
  SystemControlEntity ||--o{ HaltCleared : emits
  RequestQueue ||--o{ PlanSubmitted : emits
  PlanRequestConsumer }o--|| RequestQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlannerAgent` | `application/PlannerAgent.java` |
| `AnalystAgent` | `application/AnalystAgent.java` |
| `ArchitectAgent` | `application/ArchitectAgent.java` |
| `CoderAgent` | `application/CoderAgent.java` |
| `ReviewerAgent` | `application/ReviewerAgent.java` |
| `PlanWorkflow` | `application/PlanWorkflow.java` |
| `PlanEntity` | `application/PlanEntity.java` (state in `domain/Plan.java`, events in `domain/PlanEvent.java`) |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `RequestQueue` | `application/RequestQueue.java` |
| `PlanView` | `application/PlanView.java` |
| `PlanRequestConsumer` | `application/PlanRequestConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `StalePlanMonitor` | `application/StalePlanMonitor.java` |
| `DispatchGuardrail` | `application/DispatchGuardrail.java` |
| `SecretScrubber` | `application/SecretScrubber.java` |
| `SafetyEvaluator` | `application/SafetyEvaluator.java` |
| `PlannerTasks` | `application/PlannerTasks.java` |
| `SpecialistTasks` | `application/SpecialistTasks.java` |
| `PlanEndpoint` | `api/PlanEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `decomposeStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s (covers any specialist call, including a generous slack for a slow LLM), `decideStep` 45 s, `completeStep` 60 s. Default recovery: `maxRetries(2).failoverTo(PlanWorkflow::error)`.
- **Replan budget:** the planner may emit `Replan` at most twice in a row without a `Continue` in between; a third consecutive `Replan` is treated as `Fail`.
- **Failure budget:** the planner may emit `Continue` on the same `(specialist, subtask)` at most three times; a fourth attempt is treated as `Fail`.
- **Halt poll:** every `checkHaltStep` reads `SystemControlEntity.get` synchronously — no caching. An operator halt arriving during a `dispatchStep` lets the in-flight sub-task finish; the loop exits at the next `checkHaltStep`.
- **Idempotency:** `PlanEndpoint.submit` uses `(prompt, requestedBy)` over a 10 s window to dedupe `POST /api/plans`.
- **Stuck detection:** `StalePlanMonitor` ticks every 30 s; `PlanFailedTimeout` is non-fatal to other plans. The workflow's `decideStep` checks the entity's status and exits if it reads `STUCK`.
- **Sanitizer determinism:** `SecretScrubber.scrub` is pure; it never inspects external state. The same input always yields the same scrubbed output, keeping `ProgressEntry` events deterministic and replayable.
