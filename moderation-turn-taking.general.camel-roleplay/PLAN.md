# Implementation Plan — `camel-roleplay`

The architecture this blueprint resolves to once [`SPEC.md`](./SPEC.md) is run through `/akka:specify` → `/akka:plan`. The four mermaid diagrams below render on the Architecture tab of the generated UI; they use the Akka theme variables plus the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

---

## 1. Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#141414','primaryBorderColor':'#E6E600','primaryTextColor':'#ffffff',
  'lineColor':'#888','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans, sans-serif'
}}}%%
flowchart TB
  sim[TaskSimulator<br/>TimedAction]:::timed
  queue[InboundTaskQueue<br/>EventSourcedEntity]:::entity
  ctrl[SystemControl<br/>KeyValueEntity]:::kve
  cons[TaskRequestConsumer<br/>Consumer]:::consumer
  wf[CollaborationWorkflow<br/>Workflow — the Coordinator]:::workflow
  user[UserAgent<br/>AutonomousAgent]:::agent
  asst[AssistantAgent<br/>AutonomousAgent]:::agent
  coord[CoordinatorAgent<br/>AutonomousAgent]:::agent
  ent[CollaborationEntity<br/>EventSourcedEntity]:::entity
  eval[SolutionEvaluator<br/>Consumer]:::consumer
  view[CollaborationsView<br/>View]:::view
  mon[StalledCollaborationMonitor<br/>TimedAction]:::timed
  api[CollaborationEndpoint<br/>HttpEndpoint]:::endpoint
  app[AppEndpoint<br/>HttpEndpoint]:::endpoint

  sim -. drips canned task .-> queue
  api -- start --> queue
  queue -. TaskQueued .-> cons
  cons -- reads halt --> ctrl
  cons -- start --> wf
  wf -- USER_TURN --> user
  wf -- ASSISTANT_TURN --> asst
  wf -- COORDINATE --> coord
  wf -- recordTurn / conclude --> ent
  ent -. CollaborationConcluded .-> eval
  eval -- recordEvaluation --> ent
  ent -. events .-> view
  mon -- getAllCollaborations --> view
  mon -- markEscalated --> ent
  api -- list / single / sse --> view
  api -- halt / resume --> ctrl
  app -- serves --> app

  classDef agent fill:#3a1a1a,stroke:#E0533D,color:#fff;
  classDef workflow fill:#2a2410,stroke:#E6E600,color:#fff;
  classDef entity fill:#10241f,stroke:#3DBE8B,color:#fff;
  classDef kve fill:#10241f,stroke:#3DBE8B,color:#fff;
  classDef view fill:#101d2a,stroke:#3D8BBE,color:#fff;
  classDef consumer fill:#1d1a2a,stroke:#8B6FE0,color:#fff;
  classDef timed fill:#241a10,stroke:#BE8B3D,color:#fff;
  classDef endpoint fill:#1a1a1a,stroke:#888,color:#fff;
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions; dotted arrows are scheduled ticks.

## 2. Interaction sequence — one collaboration

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#141414','primaryBorderColor':'#E6E600','primaryTextColor':'#ffffff',
  'lineColor':'#888','actorTextColor':'#ffffff','noteTextColor':'#ffffff',
  'fontFamily':'Instrument Sans, sans-serif'
}}}%%
sequenceDiagram
  participant U as User / Simulator
  participant Q as InboundTaskQueue
  participant C as TaskRequestConsumer
  participant W as CollaborationWorkflow
  participant UA as UserAgent
  participant AA as AssistantAgent
  participant CA as CoordinatorAgent
  participant E as CollaborationEntity
  participant S as SolutionEvaluator

  U->>Q: enqueueTask(taskDescription, assistantRole, userRole)
  Q-->>C: TaskQueued
  C->>W: start(collaboration)
  W->>E: start -> CollaborationStarted
  loop up to 12 rounds
    W->>UA: runSingleTask(USER_TURN)
    UA-->>W: DialogueTurn (message, intent, taskComplete)
    W->>E: recordTurn(USER)
    W->>AA: runSingleTask(ASSISTANT_TURN)
    AA-->>W: DialogueTurn (message, intent, taskComplete)
    W->>E: recordTurn(ASSISTANT)
    W->>CA: runSingleTask(COORDINATE)
    CA-->>W: CoordinatorDecision (verdict)
    Note over W,CA: SOLVED or IMPASSE ends the loop;<br/>CONTINUE advances the round
  end
  W->>E: conclude -> CollaborationConcluded
  E-->>S: CollaborationConcluded
  S->>E: recordEvaluation(score, notes)
```

## 3. State machine — `CollaborationEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#141414','primaryBorderColor':'#E6E600','primaryTextColor':'#ffffff',
  'lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc','fontFamily':'Instrument Sans, sans-serif'
}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> COLLABORATING: CollaborationStarted
  COLLABORATING --> COLLABORATING: TurnRecorded / RoundAdvanced
  COLLABORATING --> CONCLUDED: CollaborationConcluded
  COLLABORATING --> ESCALATED: CollaborationEscalated (stalled > 3 min)
  CONCLUDED --> CONCLUDED: OutcomeEvaluated
  CONCLUDED --> [*]
  ESCALATED --> [*]
```

`CONCLUDED` carries an `outcome` of `SOLVED` or `IMPASSE`; the enum stays four-valued so no view query indexes it.

## 4. Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#141414','primaryBorderColor':'#E6E600','primaryTextColor':'#ffffff',
  'lineColor':'#888','fontFamily':'Instrument Sans, sans-serif'
}}}%%
erDiagram
  COLLABORATION ||--o{ TURN_LINE : records
  COLLABORATION ||--o{ COLLABORATION_EVENT : emits
  COLLABORATION ||--|| COLLABORATIONS_VIEW_ROW : projects
  INBOUND_TASK_QUEUE ||--o{ QUEUE_EVENT : emits
  SYSTEM_CONTROL ||--|| HALT_FLAG : holds

  COLLABORATION {
    string id
    string taskDescription
    string assistantRole
    string userRole
    enum status
    int currentRound
    string solutionSummary
    double outcomeScore
  }
  TURN_LINE {
    int round
    string agent
    string message
    boolean taskComplete
  }
  COLLABORATION_EVENT {
    string variant
    instant at
  }
```

## 5. Component table

| Component | Kind | File | Purpose |
|---|---|---|---|
| `AssistantAgent` | AutonomousAgent | `application/AssistantAgent.java` | Task-solving dialogue turn per round; returns `DialogueTurn`. |
| `UserAgent` | AutonomousAgent | `application/UserAgent.java` | Instruction-giving dialogue turn per round; returns `DialogueTurn`. |
| `CoordinatorAgent` | AutonomousAgent | `application/CoordinatorAgent.java` | Adjudicates each round; returns `CoordinatorDecision`. |
| `CollaborationTasks` | task definitions | `application/CollaborationTasks.java` | `ASSISTANT_TURN`, `USER_TURN`, `COORDINATE`. |
| `CollaborationWorkflow` | Workflow | `application/CollaborationWorkflow.java` | Turn-taking loop and convergence routing. |
| `CollaborationEntity` | EventSourcedEntity | `application/CollaborationEntity.java` | Per-collaboration durable state. |
| `InboundTaskQueue` | EventSourcedEntity | `application/InboundTaskQueue.java` | Records each task request. |
| `SystemControl` | KeyValueEntity | `application/SystemControl.java` | Operator halt flag. |
| `CollaborationsView` | View | `application/CollaborationsView.java` | Row type `Collaboration`; `getAllCollaborations` + stream. |
| `TaskRequestConsumer` | Consumer | `application/TaskRequestConsumer.java` | Starts a workflow per queued task. |
| `SolutionEvaluator` | Consumer | `application/SolutionEvaluator.java` | Scores each concluded collaboration. |
| `TaskSimulator` | TimedAction | `application/TaskSimulator.java` | Drips a canned task every 30 s. |
| `StalledCollaborationMonitor` | TimedAction | `application/StalledCollaborationMonitor.java` | Escalates collaborations running > 3 min. |
| `CollaborationEndpoint` | HttpEndpoint | `api/CollaborationEndpoint.java` | `/api/*` HTTP + SSE + metadata. |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` | Serves `/` and `/app/*`. |
| `Bootstrap` | service-setup | `Bootstrap.java` | Schedules the two TimedActions. |

Domain records live in `domain/`: `Collaboration`, `CollaborationStatus`, `CollaborationEvent`, `TurnLine`. Agent result records (`DialogueTurn`, `CoordinatorDecision`) live in `application/`.

Akka component count: **2 http-endpoint · 2 timed-action · 1 view · 1 workflow · 1 service-setup · 3 autonomous-agent · 2 consumer · 2 event-sourced-entity · 1 key-value-entity**.

## 6. Concurrency notes

- **Step timeouts.** `userTurnStep`, `assistantTurnStep`, `coordinateStep`, and `concludeStep` each call an agent, so every one overrides the 5 s default to 60 s (Lesson 4). `WorkflowSettings` is the nested `Workflow.WorkflowSettings` — no import (Lesson 5).
- **Step recovery.** `defaultStepRecovery(maxRetries(2).failoverTo(CollaborationWorkflow::error))`; the `error` step writes a `CollaborationConcluded` with `outcome = IMPASSE` so a stuck collaboration always reaches a terminal state.
- **Round cap.** The round counter lives on `CollaborationEntity` (incremented by `advanceRound`/`RoundAdvanced`), not only in workflow state, so a workflow restart resumes from the persisted round and cannot exceed twelve rounds.
- **Idempotency.** The workflow id is the collaboration id; `TaskRequestConsumer` derives a deterministic workflow id from the queue event sequence so a redelivered queue event does not start a duplicate collaboration.
- **Convergence is driven by the Coordinator.** The Coordinator evaluates whether the assistant's latest turn constitutes a complete answer and whether the user agent signals acceptance; it does not apply numeric arithmetic. The output guardrail validates that a `SOLVED` verdict always carries a non-empty `solutionSummary`.
- **Halt is a read, not a lock.** `TaskRequestConsumer` reads `SystemControl.isHalted` before starting work; in-flight collaborations are never interrupted by a halt — only new starts are gated.
- **No saga rollback needed.** Nothing in the runs-out-of-the-box form has an external irreversible side effect; a failed `concludeStep` simply fails over to the `error` step, which concludes the collaboration as `IMPASSE`.
