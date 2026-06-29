# PLAN — sentiment-monitor

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab.

---

## Component graph

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {
  'primaryColor': '#0e1e2a',
  'primaryTextColor': '#7EC8E3',
  'primaryBorderColor': '#7EC8E3',
  'lineColor': '#aab3bd',
  'secondaryColor': '#1c1330',
  'tertiaryColor': '#1f1900'
}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Poller[CommentPoller]:::ta
  Queue[CommentQueue]:::ese
  Consumer[SentimentScoringConsumer]:::cons
  Scorer[SentimentScoringAgent]:::agent
  Analyst[TrendAnalysisAgent]:::agent
  ScoreWF[SentimentScoringWorkflow]:::wf
  AlertWF[AlertDispatchWorkflow]:::wf
  Thread[IssueThreadEntity]:::ese
  EvalEnt[SentimentEvalEntity]:::ese
  View[ThreadView]:::view
  EvalRunner[SentimentEvalRunner]:::ta
  API[SentimentEndpoint]:::ep
  App[AppEndpoint]:::ep

  Poller -.->|every 20s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start per comment| ScoreWF
  ScoreWF -->|call| Scorer
  ScoreWF -->|emit CommentScored| Thread
  ScoreWF -.->|if CRITICAL| AlertWF
  AlertWF -->|call| Analyst
  AlertWF -->|guardrail + dispatch| Thread
  Thread -.->|projects| View
  API -->|silence/resolve| Thread
  API -->|query/SSE| View
  EvalRunner -.->|every 60m| EvalEnt
```

## Interaction sequence — J1 + J2

```mermaid
sequenceDiagram
  autonumber
  participant P as CommentPoller
  participant Q as CommentQueue
  participant C as SentimentScoringConsumer
  participant SW as SentimentScoringWorkflow
  participant SA as SentimentScoringAgent
  participant T as IssueThreadEntity
  participant AW as AlertDispatchWorkflow
  participant TA as TrendAnalysisAgent
  participant U as User (UI)
  participant API as SentimentEndpoint

  P->>Q: emit CommentReceived
  Q->>C: CommentReceived
  C->>SW: start(commentId, comment)
  SW->>SA: score(comment)
  SA-->>SW: SentimentScore(-4, "high", "Angry complaint")
  SW->>T: emit CommentScored → TrendUpdated
  T-->>SW: CriticalTrendDetected (5 consecutive negatives)
  SW->>AW: start(issueId)
  AW->>AW: validateChannelStep (guardrail check)
  AW->>TA: summarise(trendWindow)
  TA-->>AW: AlertSummary
  AW->>T: emit AlertDispatched
  Note over T,U: Alert goes to Slack (simulated)
  U->>API: POST /threads/{id}/silence
  API->>T: emit AlertSilenced
  T-->>API: updated IssueThreadRow
```

## State machine — `IssueThreadEntity`

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {
  'primaryColor': '#1a1c20',
  'primaryTextColor': '#e6edf3',
  'primaryBorderColor': '#30363d',
  'lineColor': '#aab3bd',
  'transitionLabelColor': '#cccccc'
}}}%%
stateDiagram-v2
  [*] --> ACTIVE : ThreadOpened
  ACTIVE --> ACTIVE : CommentScored (trend STABLE or DECLINING)
  ACTIVE --> CRITICAL_ACTIVE : CriticalTrendDetected
  CRITICAL_ACTIVE --> ACTIVE : TrendUpdated (trend recovered)
  CRITICAL_ACTIVE --> SILENCED : AlertSilenced
  ACTIVE --> SILENCED : AlertSilenced
  SILENCED --> ACTIVE : ThreadResolved (re-activates)
  ACTIVE --> RESOLVED : ThreadResolved
  SILENCED --> RESOLVED : ThreadResolved
  CRITICAL_ACTIVE --> RESOLVED : ThreadResolved
  RESOLVED --> [*]
```

## Entity model

```mermaid
erDiagram
  IssueThreadEntity ||--o{ CommentScored : emits
  IssueThreadEntity ||--o{ TrendUpdated : emits
  IssueThreadEntity ||--o{ CriticalTrendDetected : emits
  IssueThreadEntity ||--o{ AlertDispatched : emits
  IssueThreadEntity ||--o{ AlertSilenced : emits
  IssueThreadEntity ||--o{ ThreadResolved : emits
  IssueThreadEntity ||--o{ ThreadOpened : emits
  ThreadView }o--|| IssueThreadEntity : projects
  ThreadView }o--|| SentimentEvalEntity : projects
  CommentQueue ||--o{ CommentReceived : emits
  SentimentScoringConsumer }o--|| CommentQueue : subscribes
  SentimentEvalEntity ||--o{ DriftEvalRecorded : emits
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `CommentPoller` | `application/CommentPoller.java` |
| `CommentQueue` | `application/CommentQueue.java` |
| `SentimentScoringConsumer` | `application/SentimentScoringConsumer.java` |
| `SentimentScoringAgent` | `application/SentimentScoringAgent.java` |
| `TrendAnalysisAgent` | `application/TrendAnalysisAgent.java` |
| `SentimentScoringWorkflow` | `application/SentimentScoringWorkflow.java` |
| `AlertDispatchWorkflow` | `application/AlertDispatchWorkflow.java` |
| `IssueThreadEntity` | `application/IssueThreadEntity.java` (state in `domain/IssueThread.java`, events in `domain/IssueThreadEvent.java`) |
| `SentimentEvalEntity` | `application/SentimentEvalEntity.java` |
| `ThreadView` | `application/ThreadView.java` |
| `SentimentEvalRunner` | `application/SentimentEvalRunner.java` |
| `SentimentEndpoint` | `api/SentimentEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: scorer 15 s (neutral fallback on timeout), trend analyst 30 s (skip dispatch on timeout).
- **Guardrail gate**: `AlertDispatchWorkflow.validateChannelStep` reads the allowlist from application.conf; rejection is a structured error, not an exception — the workflow records the rejection without crashing.
- **Idempotency**: each `SentimentScoringWorkflow` uses `commentId` as the workflow id; each `AlertDispatchWorkflow` uses `issueId + alertEpochSecond` to prevent double dispatch within the same trend window.
- **Eval sampling**: per tick, `SentimentEvalRunner` picks up to 10 scored comments with ground-truth labels, oldest-first, from the JSONL resource file.
