# PLAN — code-team-team

Architectural sketch for the Game Builder Team sample. Four mermaid diagrams + the component table.

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

  Sim[BriefSimulator]:::ta -. every 30s .-> Queue[InboundBriefQueue]:::ese
  EP[BuildEndpoint]:::ep --> Queue
  Queue -. BriefQueued .-> Cons[BriefConsumer]:::cons
  Cons --> WF[GameBuildWorkflow]:::wf
  WF --> Eng[EngineerAgent]:::agent
  WF --> Qa[QaAgent]:::agent
  WF --> Chief[ChiefQaAgent]:::agent
  WF --> Build[BuildEntity]:::ese
  Build -. events .-> View[BuildsView]:::view
  EP --> View
  App[AppEndpoint]:::ep --> Static[static-resources]
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions; dotted arrows are scheduled ticks.

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  participant U as User / Simulator
  participant Q as InboundBriefQueue
  participant C as BriefConsumer
  participant W as GameBuildWorkflow
  participant E as EngineerAgent
  participant A as QaAgent
  participant H as ChiefQaAgent
  participant B as BuildEntity
  U->>Q: enqueueBrief(brief)
  Q-->>C: BriefQueued
  C->>W: start(buildId)
  W->>E: engineerStep
  E-->>W: GameCode
  W->>B: recordEngineered
  W->>A: qaStep (guarded sandbox run)
  A-->>W: QaReport
  W->>B: recordQa
  Note over W: ci-gate — ship only if QaReport.passed
  W->>H: chiefReviewStep
  H-->>W: ShipDecision
  alt passed and shipped
    W->>B: ship
  else
    W->>B: reject(reworkReason)
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> QUEUED
  QUEUED --> ENGINEERED: GameEngineered
  ENGINEERED --> QA: QaCompleted
  QA --> SHIPPED: BuildShipped
  QA --> REJECTED: BuildRejected
  SHIPPED --> [*]
  REJECTED --> [*]
```

## Entity model

```mermaid
erDiagram
  INBOUND_BRIEF_QUEUE ||--o{ BRIEF_QUEUED : emits
  BUILD_ENTITY ||--o{ GAME_ENGINEERED : emits
  BUILD_ENTITY ||--o{ QA_COMPLETED : emits
  BUILD_ENTITY ||--o{ BUILD_SHIPPED : emits
  BUILD_ENTITY ||--o{ BUILD_REJECTED : emits
  BUILD_ENTITY ||--|| BUILDS_VIEW : projects
  BUILDS_VIEW {
    string id
    string status
    string filename
    int qaScore
  }
```

## Component table

| Component | Path (generated) |
|---|---|
| EngineerAgent | `application/EngineerAgent.java` |
| QaAgent | `application/QaAgent.java` |
| ChiefQaAgent | `application/ChiefQaAgent.java` |
| BuildTasks | `application/BuildTasks.java` |
| GameBuildWorkflow | `application/GameBuildWorkflow.java` |
| BriefConsumer | `application/BriefConsumer.java` |
| BriefSimulator | `application/BriefSimulator.java` |
| BuildsView | `application/BuildsView.java` |
| BuildEntity | `domain/BuildEntity.java` |
| InboundBriefQueue | `domain/InboundBriefQueue.java` |
| BuildEndpoint | `api/BuildEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts.** `engineerStep`, `qaStep`, and `chiefReviewStep` each set `stepTimeout(ofSeconds(60))` because they call agents; the default 5 s timeout would expire mid-call (Lesson 4). `defaultStepRecovery(maxRetries(2).failoverTo(error))` routes exhausted retries to a terminal error step.
- **Idempotency.** One `buildId` (UUID) per queued brief; the workflow keys all entity commands by that id, so a replayed `BriefQueued` starts a distinct build rather than corrupting an existing one.
- **No compensation needed.** The pipeline is forward-only and the publish target is in-process; a rejected build is a terminal state, not a rollback. The ci-gate (ship only when QA passed) lives in `chiefReviewStep`, so there is no irreversible action to compensate.
- **View indexing.** `BuildsView` exposes a single `getAllBuilds` query with no `WHERE status` clause; status filtering happens client-side in the endpoint (Lesson 2).
