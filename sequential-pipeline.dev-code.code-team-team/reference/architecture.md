# Architecture — Game Builder Team

A sequential pipeline. One inbound brief produces one build that moves through three agent stages in a fixed order — engineer, QA, chief QA — and ends shipped or rejected. The diagrams below are the source the generated system renders on the Architecture tab.

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

A brief enters through either the `BuildEndpoint` (user POST) or the `BriefSimulator` timer. Both write to `InboundBriefQueue`. The `BriefConsumer` reacts to each `BriefQueued` event and starts one `GameBuildWorkflow`. The workflow drives the three agents and writes lifecycle events to `BuildEntity`, which the `BuildsView` projects for the endpoint to query and stream.

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

The QA step is the in-band evaluation point: its `QaReport` carries the score that control E1 records, and the test outcome feeds the ci-gate (A1) that the chief-review step enforces. The guardrail (G1) fires inside `qaStep`, before the sandbox runs the candidate source.

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

The lifecycle is forward-only. `SHIPPED` and `REJECTED` are terminal; there is no rollback because the publish target is in-process and a rejected build is simply a terminal state.

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

`InboundBriefQueue` and `BuildEntity` are the two event-sourced entities. `BuildsView` is the single read model; it exposes one `getAllBuilds` query with no `WHERE status` clause, so status filtering happens client-side in the endpoint (Akka cannot auto-index enum columns).
