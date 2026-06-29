# PLAN — collab-research-team

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

  Coord[ResearchCoordinator]:::agent
  Res[ResearcherAgent]:::agent
  Synth[SynthesisAgent]:::agent

  Plan[PlanningWorkflow]:::wf
  ResWF[ResearcherWorkflow]:::wf
  SynthWF[SynthesisWorkflow]:::wf

  Question[QuestionEntity]:::ese
  TaskE[ResearchTaskEntity]:::ese
  Mailbox[ResearcherMailbox]:::ese
  Intake[IntakeQueue]:::ese
  Control[OperatorControl]:::kve
  Board[TaskBoardView]:::view
  Consumer[QuestionRequestConsumer]:::cons
  Sim[QuestionSimulator]:::ta
  CompMon[QuestionCompletionMonitor]:::ta
  Stuck[StuckTaskMonitor]:::ta
  API[ResearchEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit question| Intake
  Sim -.->|every 90s| Intake
  Intake -.->|subscribes| Consumer
  Consumer -->|create + start| Question
  Consumer -->|start workflow| Plan
  Plan -->|call| Coord
  Plan -->|create one per sub-topic| TaskE
  TaskE -.->|projects| Board
  ResWF -->|poll board| Board
  ResWF -->|atomic claim| TaskE
  ResWF -->|call| Res
  ResWF -->|post message| Mailbox
  ResWF -->|read flag| Control
  CompMon -.->|every 30s| Question
  CompMon -->|start when all done| SynthWF
  SynthWF -->|call| Synth
  SynthWF -->|eval per conclusion| Question
  Stuck -.->|every 30s| TaskE
  API -->|halt / resume| Control
  API -->|reply| Mailbox
  API -->|approve / reject| Question
  API -->|query / SSE| Board
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions and scheduled ticks. `ResearcherAgent` is one agent class run as several instances (`researcher-1`, `researcher-2`, `researcher-3`); each instance is driven by its own `ResearcherWorkflow`. `SynthesisWorkflow` is started per question by `QuestionCompletionMonitor` once all sub-topic tasks are done.

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ResearchEndpoint
  participant Q as IntakeQueue
  participant C as QuestionRequestConsumer
  participant PW as PlanningWorkflow
  participant Coord as ResearchCoordinator
  participant T as ResearchTaskEntity
  participant RW as ResearcherWorkflow
  participant R as ResearcherAgent
  participant V as TaskBoardView
  participant Mon as QuestionCompletionMonitor
  participant SW as SynthesisWorkflow
  participant S as SynthesisAgent

  U->>API: POST /api/questions {title, context}
  API->>Q: submitQuestion(brief)
  API-->>U: 202 {questionId}
  Q->>C: QuestionSubmitted
  C->>PW: start({questionId})
  PW->>Coord: plan(question)
  Coord-->>PW: ResearchPlan{subTopics}
  PW->>T: createTask (one per sub-topic, status OPEN)
  T-->>V: task rows
  Note over RW: researcher loops already polling
  RW->>V: getAllTasks
  V-->>RW: OPEN task
  RW->>T: claim(researcher-1)
  T-->>RW: TaskClaimed (won the race)
  RW->>R: research(task)
  Note over R: every response passes the after-llm-response guardrail (G1)
  R-->>RW: FindingsReport{sources, summary}
  RW->>T: submitFindings then complete -> DONE
  T-->>V: row DONE
  V-->>U: SSE update
  Mon->>Mon: all tasks DONE for question?
  Mon->>SW: start({questionId})
  SW->>S: synthesize(allFindings)
  S-->>SW: ResearchReport{conclusions}
  SW->>SW: eval each conclusion (E1)
  SW->>Q: complete(report)
  Q-->>U: SSE question COMPLETED
```

## State machine — `ResearchTaskEntity`

```mermaid
stateDiagram-v2
  [*] --> OPEN
  OPEN --> CLAIMED: claim (atomic, single winner)
  CLAIMED --> IN_PROGRESS: start
  CLAIMED --> OPEN: release (stuck > 3 min)
  IN_PROGRESS --> FINDINGS_READY: submitFindings
  IN_PROGRESS --> BLOCKED: coordination request raised
  IN_PROGRESS --> BLOCKED: guardrail refusal (too few sources)
  IN_PROGRESS --> OPEN: release (stuck > 3 min)
  FINDINGS_READY --> DONE: complete
  BLOCKED --> OPEN: coordination reply / operator reopen
  DONE --> [*]
```

## Entity model

```mermaid
erDiagram
  ResearchTaskEntity ||--o{ TaskCreated : emits
  ResearchTaskEntity ||--o{ TaskClaimed : emits
  ResearchTaskEntity ||--o{ TaskStarted : emits
  ResearchTaskEntity ||--o{ FindingsSubmitted : emits
  ResearchTaskEntity ||--o{ TaskCompleted : emits
  ResearchTaskEntity ||--o{ TaskBlocked : emits
  ResearchTaskEntity ||--o{ TaskReleased : emits
  TaskBoardView }o--|| ResearchTaskEntity : projects
  QuestionEntity ||--o{ QuestionCreated : emits
  QuestionEntity ||--o{ QuestionPlanned : emits
  QuestionEntity ||--o{ SynthesisStarted : emits
  QuestionEntity ||--o{ ReportReady : emits
  QuestionEntity ||--o{ ReportApproved : emits
  QuestionEntity ||--o{ ResearchTaskEntity : "owns N sub-topics"
  IntakeQueue ||--o{ QuestionSubmitted : emits
  QuestionRequestConsumer }o--|| IntakeQueue : subscribes
  ResearcherMailbox ||--o{ MessagePosted : emits
  ResearcherMailbox ||--o{ MessageReplied : emits
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ResearchCoordinator` | `application/ResearchCoordinator.java` |
| `ResearcherAgent` | `application/ResearcherAgent.java` |
| `SynthesisAgent` | `application/SynthesisAgent.java` |
| `ResearchTasks` | `application/ResearchTasks.java` |
| `CitationEvaluator` | `application/CitationEvaluator.java` |
| `PlanningWorkflow` | `application/PlanningWorkflow.java` |
| `ResearcherWorkflow` | `application/ResearcherWorkflow.java` |
| `SynthesisWorkflow` | `application/SynthesisWorkflow.java` |
| `ResearchTaskEntity` | `application/ResearchTaskEntity.java` (state in `domain/ResearchTask.java`, events in `domain/ResearchTaskEvent.java`) |
| `QuestionEntity` | `application/QuestionEntity.java` (state in `domain/Question.java`, events in `domain/QuestionEvent.java`) |
| `ResearcherMailbox` | `application/ResearcherMailbox.java` (state + events in `domain/`) |
| `IntakeQueue` | `application/IntakeQueue.java` |
| `OperatorControl` | `application/OperatorControl.java` |
| `TaskBoardView` | `application/TaskBoardView.java` |
| `QuestionRequestConsumer` | `application/QuestionRequestConsumer.java` |
| `QuestionSimulator` | `application/QuestionSimulator.java` |
| `QuestionCompletionMonitor` | `application/QuestionCompletionMonitor.java` |
| `StuckTaskMonitor` | `application/StuckTaskMonitor.java` |
| `ResearchEndpoint` | `api/ResearchEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `Bootstrap` | `Bootstrap.java` |

Akka component count: **3 autonomous-agent · 3 workflow · 4 event-sourced-entity · 1 key-value-entity · 1 view · 1 consumer · 3 timed-action · 2 http-endpoint · 1 service-setup**.

## Concurrency notes

- **Atomic claim is the whole pattern.** `ResearchTaskEntity` is a single-writer; `claim(researcherId)` emits `TaskClaimed` only when the current status is `OPEN`. Two researcher workflows that read the same `OPEN` task from the view and both call `claim` are serialised by the entity — the first wins, the second receives the already-claimed `ResearchTask` and returns to polling. No lock, no external queue.
- **Workflow step timeouts:** `PlanningWorkflow.planStep`, `ResearcherWorkflow.researchStep`, and `SynthesisWorkflow.synthesizeStep` call agents, so each sets an explicit `stepTimeout` of 90 s (Lesson 4). The default 5 s timeout would expire mid-LLM-call.
- **Idle polling:** `ResearcherWorkflow.pollStep` self-schedules a 5 s resume timer when the team is halted or no eligible `OPEN` task exists, so an idle researcher is a paused workflow, not a busy loop.
- **Release for liveness:** `StuckTaskMonitor` returns a task claimed-but-idle for more than three minutes to `OPEN`, so a researcher that fails mid-task does not strand work. `release` is a no-op unless the task is `CLAIMED` or `IN_PROGRESS`.
- **Citation eval gate:** `CitationEvaluator` is a deterministic pure function (no LLM call) applied per `ReportConclusion` in `SynthesisWorkflow.evalStep`. The same evaluator runs in tests so the eval outcome is reproducible.
- **Synthesis trigger:** `QuestionCompletionMonitor` polls every 30 s and starts `SynthesisWorkflow` once; deterministic `questionId` as workflow id makes the start idempotent if the monitor fires more than once.
- **Halt:** `OperatorControl` is read at the top of `pollStep` and inside the after-llm-response guardrail, so a halt both stops new claims and blocks any in-flight researcher response.
- **Idempotency:** deterministic `taskId = questionId + "-t" + index` makes `createTask` idempotent if `PlanningWorkflow.createTasksStep` is retried.
