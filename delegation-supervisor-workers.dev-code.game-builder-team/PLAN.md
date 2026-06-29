# PLAN — game-builder-team

Architecture sketch for the Game Builder Team system. The generated system renders these diagrams on the Architecture tab. Apply the Lesson 24 mermaid CSS overrides so state labels and edge labels render legibly.

---

## 1. Component graph

```mermaid
flowchart TB
  %%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#7EC8E3','fontFamily':'Instrument Sans'}}}%%
  sim([RequestSimulator<br/>TimedAction]) -. drips .-> queue[BuildRequestQueue<br/>EventSourcedEntity]
  gameEp[GameEndpoint<br/>HttpEndpoint] --> queue
  queue -. events .-> consumer[BuildRequestConsumer<br/>Consumer]
  consumer --> wf[GameBuildWorkflow<br/>Workflow]
  wf --> director[GameDirector<br/>AutonomousAgent]
  wf --> designer[GameDesigner<br/>AutonomousAgent]
  wf --> coder[CodeWriter<br/>AutonomousAgent]
  wf --> entity[GameProjectEntity<br/>EventSourcedEntity]
  entity -. events .-> view[GameView<br/>View]
  gameEp --> view
  gameEp --> entity
  appEp[AppEndpoint<br/>HttpEndpoint] --> static[(static-resources)]
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions; dotted arrows are scheduled ticks.

## 2. Interaction sequence

```mermaid
sequenceDiagram
  participant U as User / Simulator
  participant Q as BuildRequestQueue
  participant C as BuildRequestConsumer
  participant W as GameBuildWorkflow
  participant DI as GameDirector
  participant DE as GameDesigner
  participant CW as CodeWriter
  participant E as GameProjectEntity
  U->>Q: enqueue(idea)
  Q-->>C: BuildEnqueued
  C->>W: start(projectId, idea)
  W->>E: start
  W->>DI: delegate design
  DI->>DE: GameSpec task
  DE-->>W: GameSpec
  W->>E: recordSpec
  W->>DI: delegate code
  DI->>CW: GameCode task
  CW-->>W: GameCode
  W->>E: recordCode
  Note over W,DI: before-tool-call guardrail vets GameCode
  W->>W: SandboxRunner checks (test gate)
  alt passes
    W->>E: recordTestsPassed
    W->>E: deliver
  else fails after retries
    W->>E: recordTestsFailed
  else guardrail trips
    W->>E: block(reason)
  end
```

## 3. State machine

```mermaid
stateDiagram-v2
  [*] --> QUEUED
  QUEUED --> DESIGNING: start
  DESIGNING --> CODING: SpecDesigned
  CODING --> TESTING: CodeWritten
  TESTING --> DELIVERED: TestsPassed
  TESTING --> CODING: retry budget left
  TESTING --> TEST_FAILED: TestsFailed
  TESTING --> BLOCKED: guardrail trip
  DELIVERED --> [*]
  TEST_FAILED --> [*]
  BLOCKED --> [*]
```

## 4. Entity model

```mermaid
erDiagram
  GAME_PROJECT_ENTITY ||--o{ EVENT : emits
  GAME_PROJECT_ENTITY ||--|| GAME_VIEW : projects
  BUILD_REQUEST_QUEUE ||--o{ BUILD_ENQUEUED : emits
  GAME_PROJECT_ENTITY {
    string id
    string idea
    enum status
    json spec
    json code
    json testReport
    int testAttempts
  }
  EVENT {
    string type
  }
  GAME_VIEW {
    string id
    string status
    string idea
  }
```

## 5. Component table

| Component | Akka primitive | File path |
|---|---|---|
| `GameDirector` | AutonomousAgent | `application/GameDirector.java` |
| `GameDesigner` | AutonomousAgent | `application/GameDesigner.java` |
| `CodeWriter` | AutonomousAgent | `application/CodeWriter.java` |
| `GameBuildTasks` | task constants | `application/GameBuildTasks.java` |
| `GameBuildWorkflow` | Workflow | `application/GameBuildWorkflow.java` |
| `SandboxRunner` | helper | `application/SandboxRunner.java` |
| `GameProjectEntity` | EventSourcedEntity | `application/GameProjectEntity.java` |
| `BuildRequestQueue` | EventSourcedEntity | `application/BuildRequestQueue.java` |
| `GameView` | View | `application/GameView.java` |
| `BuildRequestConsumer` | Consumer | `application/BuildRequestConsumer.java` |
| `RequestSimulator` | TimedAction | `application/RequestSimulator.java` |
| `GameEndpoint` | HttpEndpoint | `api/GameEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |
| `GameProject`, `GameSpec`, `GameCode`, `TestReport` | records | `domain/` |

## 6. Concurrency notes

- **Step timeouts.** `designStep`, `codeStep`, and `assembleStep` each call an agent; override `settings()` with `stepTimeout(60s)` per Lesson 4. `WorkflowSettings` is nested in `Workflow` — no import.
- **Retry budget.** `testStep` retries `codeStep` up to twice when the test gate fails before settling in `TEST_FAILED`. The attempt count lives in `GameProject.testAttempts` so retries survive restarts.
- **Idempotency.** Each build is keyed by a fresh UUID assigned by `BuildRequestConsumer`; replaying a `BuildEnqueued` event with the same offset does not start a second workflow.
- **Compensation.** A guardrail trip in `testStep` is terminal — `block(reason)` writes `BuildBlocked` and the workflow ends; no run tool executes, so there is nothing to roll back.
- **View indexing.** `GameView` exposes one query (`getAllProjects`) with no enum `WHERE`; status filtering happens client-side in `GameEndpoint` (Lesson 2).
