# PLAN — web-tools-replay

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef scorer fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[SearchRunEndpoint]:::ep
  Entity[SearchRunEntity]:::ese
  Recorder[TraceRecordingConsumer]:::cons
  WF[ReplayWorkflow]:::wf
  Agent[SearchReplayAgent]:::agent
  Scorer[ReplayDriftScorer]:::scorer
  Stub[WebSearchStub]:::scorer
  View[SearchRunView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|SearchCompleted| Recorder
  Recorder -->|recordTrace| Entity
  Recorder -->|start workflow| WF
  WF -->|awaitTraceStep poll| Entity
  WF -->|replayStep runSingleTask| Agent
  Agent -->|tool calls| Stub
  Stub -->|seeded results| Agent
  Agent -->|SearchTrace| WF
  WF -->|recordReplayTrace| Entity
  WF -->|scoreStep compare| Scorer
  Scorer -->|DriftReport| WF
  WF -->|recordDrift| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as SearchRunEndpoint
  participant E as SearchRunEntity
  participant Rec as TraceRecordingConsumer
  participant W as ReplayWorkflow
  participant A as SearchReplayAgent
  participant St as WebSearchStub
  participant Sc as ReplayDriftScorer

  U->>API: POST /api/runs
  API->>E: submit(query)
  E-->>API: { runId }
  E->>E: markSearching
  API->>A: runSingleTask(queryText)
  A->>St: web_search("AI regulatory updates EU 2026")
  St-->>A: [{title, url, snippet} × 5]
  A-->>API: SearchTrace
  API->>E: markSearchCompleted + recordTrace(trace)
  E-.->>Rec: SearchCompleted
  Rec->>E: recordTrace(trace)
  Rec->>W: start(runId)
  W->>E: poll getRun
  E-->>W: originalTrace.isPresent()
  W->>E: markReplaying
  W->>A: runSingleTask(queryText) [replay]
  A->>St: web_search("AI regulatory updates EU 2026")
  St-->>A: [{title, url, snippet} × 5]
  A-->>W: SearchTrace [replay]
  W->>E: recordReplayTrace(replayTrace)
  W->>Sc: score(originalTrace, replayTrace)
  Sc-->>W: DriftReport{score:0}
  W->>E: recordDrift → DRIFT_CLEAR
  E-.->>U: SSE event(DRIFT_CLEAR)
```

## State machine — `SearchRunEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SEARCHING: SearchStarted
  SEARCHING --> TRACE_RECORDED: TraceRecorded
  TRACE_RECORDED --> REPLAYING: ReplayStarted
  REPLAYING --> DRIFT_CLEAR: DriftScored (score < 30)
  REPLAYING --> DRIFT_DETECTED: DriftScored (score >= 30)
  SEARCHING --> FAILED: RunFailed (agent error)
  REPLAYING --> FAILED: RunFailed (replay error)
  DRIFT_CLEAR --> [*]
  DRIFT_DETECTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  SearchRunEntity ||--o{ RunSubmitted : emits
  SearchRunEntity ||--o{ SearchStarted : emits
  SearchRunEntity ||--o{ SearchCompleted : emits
  SearchRunEntity ||--o{ TraceRecorded : emits
  SearchRunEntity ||--o{ ReplayStarted : emits
  SearchRunEntity ||--o{ DriftScored : emits
  SearchRunEntity ||--o{ RunFailed : emits
  SearchRunView }o--|| SearchRunEntity : projects
  TraceRecordingConsumer }o--|| SearchRunEntity : subscribes
  ReplayWorkflow }o--|| SearchRunEntity : reads-and-writes
  SearchReplayAgent ||--o{ SearchTrace : returns
  ReplayDriftScorer ||--o{ DriftReport : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SearchRunEndpoint` | `api/SearchRunEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SearchRunEntity` | `application/SearchRunEntity.java` (state in `domain/SearchRun.java`, events in `domain/SearchRunEvent.java`) |
| `TraceRecordingConsumer` | `application/TraceRecordingConsumer.java` |
| `ReplayWorkflow` | `application/ReplayWorkflow.java` |
| `SearchReplayAgent` | `application/SearchReplayAgent.java` (tasks in `application/SearchTasks.java`) |
| `WebSearchStub` | `application/WebSearchStub.java` |
| `ReplayDriftScorer` | `application/ReplayDriftScorer.java` |
| `SearchRunView` | `application/SearchRunView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitTraceStep` 20 s, `replayStep` 60 s, `scoreStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ReplayWorkflow::error)`. The 60 s on `replayStep` accommodates LLM latency when a real provider is used (Lesson 4).
- **Idempotency**: every workflow uses `"replay-" + runId` as the workflow id; the `TraceRecordingConsumer` Consumer is allowed to redeliver `SearchCompleted` events because `SearchRunEntity.recordTrace` is event-version-guarded — a second trace attempt against an already-recorded run is a no-op.
- **One agent per run**: the AutonomousAgent instance id is `"searcher-" + runId` for the initial search and `"replay-" + runId` for the replay, each with its own conversation context. `maxIterationsPerTask(3)` caps retries.
- **Drift is deterministic**: `ReplayDriftScorer` runs in-process inside `scoreStep`. Same two traces always produce the same score — no LLM, no external service. This upholds the single-agent invariant.
- **Manual replay**: `GET /api/runs/{id}/replay` starts a new `ReplayWorkflow` instance with a versioned id `"replay-" + runId + "-" + epochSecond`. Only valid when the run is in `TRACE_RECORDED`, `DRIFT_CLEAR`, or `DRIFT_DETECTED` state. The entity transitions back to `REPLAYING` on `ReplayStarted`.
- **No saga / no compensation**: every step is either a pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
