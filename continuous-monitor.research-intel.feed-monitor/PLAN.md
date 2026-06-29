# PLAN — feed-monitor

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

  Poller[FeedPoller]:::ta
  Entity[FeedItemEntity]:::ese
  Summarizer[SummaryAgent]:::agent
  Classifier[ClassifierAgent]:::agent
  WF[FeedWorkflow]:::wf
  Notifier[SlackNotifier]:::cons
  View[FeedView]:::view
  EvalRunner[EvalRunner]:::ta
  API[FeedEndpoint]:::ep
  App[AppEndpoint]:::ep

  Poller -.->|every 60s| Entity
  Entity -.->|on ItemReceived| WF
  WF -->|call| Summarizer
  WF -->|call| Classifier
  WF -->|emit events| Entity
  Entity -.->|on NotifyRequested| Notifier
  Notifier -->|guardrail check + postToSlack stub| Entity
  Entity -.->|projects| View
  API -->|override/suppress| Entity
  API -->|query/SSE| View
  EvalRunner -.->|every 30m| Entity
```

## Interaction sequence — J1

```mermaid
sequenceDiagram
  autonumber
  participant P as FeedPoller
  participant E as FeedItemEntity
  participant W as FeedWorkflow
  participant S as SummaryAgent
  participant C as ClassifierAgent
  participant N as SlackNotifier
  participant U as User (UI)
  participant API as FeedEndpoint

  P->>E: emit ItemReceived
  E->>W: start({itemId, raw})
  W->>S: summarize(raw)
  S-->>W: SummaryResult
  W->>E: emit ItemSummarized
  W->>C: classify(summary, raw)
  C-->>W: NOTIFY / targetChannel=#research-intel
  W->>E: emit ItemClassified + NotifyRequested
  Note over E,N: SlackNotifier Consumer picks up NotifyRequested
  N->>N: guardrail: channel in allow-list?
  N->>E: emit ItemPosted (stub postToSlack called)
  Note over E,U: Item status = POSTED
  U->>API: GET /api/feed/{id}
  API-->>U: FeedItemState with post + evalScore (once EvalRunner runs)
```

## State machine — `FeedItemEntity`

```mermaid
stateDiagram-v2
  %%{init: {"theme": "dark", "themeVariables": {"transitionLabelColor": "#cccccc"}}}%%
  [*] --> RECEIVED
  RECEIVED --> SUMMARIZED: ItemSummarized
  SUMMARIZED --> CLASSIFIED: ItemClassified
  CLASSIFIED --> NOTIFY_REQUESTED: classification = NOTIFY
  CLASSIFIED --> SUPPRESSED: classification = SUPPRESS
  CLASSIFIED --> PENDING_REVIEW: classification = REVIEW
  NOTIFY_REQUESTED --> POSTED: ItemPosted (guardrail passed)
  NOTIFY_REQUESTED --> FAILED: ItemFailed (guardrail rejected)
  PENDING_REVIEW --> NOTIFY_REQUESTED: ReviewOverridden (notify)
  PENDING_REVIEW --> SUPPRESSED: ReviewOverridden (suppress)
  POSTED --> [*]
  SUPPRESSED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  FeedItemEntity ||--o{ ItemReceived : emits
  FeedItemEntity ||--o{ ItemSummarized : emits
  FeedItemEntity ||--o{ ItemClassified : emits
  FeedItemEntity ||--o{ NotifyRequested : emits
  FeedItemEntity ||--o{ ItemPosted : emits
  FeedItemEntity ||--o{ ItemSuppressed : emits
  FeedItemEntity ||--o{ ItemPendingReview : emits
  FeedItemEntity ||--o{ ItemFailed : emits
  FeedItemEntity ||--o{ ReviewOverridden : emits
  FeedItemEntity ||--o{ EvalScored : emits
  FeedView }o--|| FeedItemEntity : projects
  SlackNotifier }o--|| FeedItemEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `FeedPoller` | `application/FeedPoller.java` |
| `FeedItemEntity` | `application/FeedItemEntity.java` (state in `domain/FeedItemState.java`, events in `domain/FeedItemEvent.java`) |
| `SummaryAgent` | `application/SummaryAgent.java` |
| `ClassifierAgent` | `application/ClassifierAgent.java` |
| `FeedWorkflow` | `application/FeedWorkflow.java` |
| `SlackNotifier` | `application/SlackNotifier.java` |
| `FeedView` | `application/FeedView.java` |
| `EvalRunner` | `application/EvalRunner.java` |
| `FeedEndpoint` | `api/FeedEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: summarize 20 s, classify 10 s. On timeout, emit `ItemFailed(reason="step-timeout")`.
- **Guardrail check**: `SlackNotifier` synchronously validates channel before the stub call; on rejection, transitions to FAILED immediately.
- **Idempotency**: every workflow uses `itemId` as the workflow id so duplicate `ItemReceived` events fold into one workflow.
- **Review override**: `FeedEndpoint.override` and `FeedEndpoint.suppress` can advance PENDING_REVIEW items; they are the only callers of `recordReviewOverride`.
- **Eval sampling**: per tick, EvalRunner picks up to 5 POSTED items with no `evalScore`, oldest-first.
