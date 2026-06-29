# PLAN — gitwiki

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
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[WikiEndpoint]:::ep
  Entity[PageEntity]:::ese
  Consumer[GitPushConsumer]:::cons
  WF[WikiUpdateWorkflow]:::wf
  Agent[WikiEditorAgent]:::agent
  Guard[GitPushGuardrail]:::guard
  ConfigGate[StartupConfigGate]:::guard
  View[PageView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|configCheckStep| ConfigGate
  WF -->|editStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|CommitDraft| WF
  WF -->|markCommitReady| Entity
  Entity -.->|CommitReady| Consumer
  Consumer -->|check| Guard
  Guard -.->|pass / reject| Consumer
  Consumer -->|recordPushResult| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as WikiEndpoint
  participant E as PageEntity
  participant W as WikiUpdateWorkflow
  participant A as WikiEditorAgent
  participant G as GitPushGuardrail
  participant C as GitPushConsumer

  U->>API: POST /api/updates
  API->>E: submit(request)
  E-->>API: { updateId }
  API->>W: start(updateId)
  W->>W: configCheckStep (StartupConfigGate.isHealthy)
  W->>E: markEditing
  W->>A: runSingleTask(changes + attachment)
  A->>G: before-tool-call(push params)
  G-->>A: pass
  A-->>W: CommitDraft
  W->>E: markCommitReady(draft)
  E-.->>C: CommitReady
  C->>G: check(pagePath, targetRef, author)
  G-->>C: pass
  C->>E: markPushStarted
  C->>C: GitHub API push
  C->>E: recordPushResult(SUCCESS, sha)
  E-.->>U: SSE event(PUSHED)
```

## State machine — `PageEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> EDITING: EditingStarted
  EDITING --> COMMIT_READY: CommitReady
  COMMIT_READY --> PUSH_IN_PROGRESS: PushStarted
  PUSH_IN_PROGRESS --> PUSHED: PushCompleted
  PUSH_IN_PROGRESS --> PUSH_REJECTED: PushRejected (guardrail)
  PUSH_IN_PROGRESS --> CONFLICT: PushCompleted (non-fast-forward)
  EDITING --> FAILED: UpdateFailed (agent error)
  SUBMITTED --> FAILED: UpdateFailed (config gate)
  PUSHED --> [*]
  PUSH_REJECTED --> [*]
  CONFLICT --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  PageEntity ||--o{ UpdateSubmitted : emits
  PageEntity ||--o{ EditingStarted : emits
  PageEntity ||--o{ CommitReady : emits
  PageEntity ||--o{ PushStarted : emits
  PageEntity ||--o{ PushCompleted : emits
  PageEntity ||--o{ PushRejected : emits
  PageEntity ||--o{ UpdateFailed : emits
  PageView }o--|| PageEntity : projects
  GitPushConsumer }o--|| PageEntity : subscribes
  WikiUpdateWorkflow }o--|| PageEntity : reads-and-writes
  WikiEditorAgent ||--o{ CommitDraft : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `WikiEndpoint` | `api/WikiEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `PageEntity` | `application/PageEntity.java` (state in `domain/CommitOutcome.java`, events in `domain/PageEvent.java`) |
| `GitPushConsumer` | `application/GitPushConsumer.java` |
| `WikiUpdateWorkflow` | `application/WikiUpdateWorkflow.java` |
| `WikiEditorAgent` | `application/WikiEditorAgent.java` (tasks in `application/WikiTasks.java`) |
| `GitPushGuardrail` | `application/GitPushGuardrail.java` |
| `StartupConfigGate` | `application/StartupConfigGate.java` |
| `PageView` | `application/PageView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `configCheckStep` 5 s, `editStep` 60 s, `awaitPushStep` 30 s, `error` 5 s. Default step recovery `maxRetries(1).failoverTo(WikiUpdateWorkflow::error)`. The 60 s on `editStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"update-" + updateId` as the workflow id. `GitPushConsumer` is allowed to redeliver `CommitReady` events because `PageEntity.markCommitReady` is event-version-guarded — a second delivery against an already-committed update is a no-op.
- **One agent per update**: the AutonomousAgent instance id is `"editor-" + updateId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(2)` caps guardrail-triggered retries at 2.
- **Guardrail-driven rejection**: when `GitPushGuardrail` rejects a push attempt, the rejection is recorded synchronously; `GitPushConsumer` calls `PageEntity.recordPushResult(PushResult.rejected(reason))`. The entity transitions to `PUSH_REJECTED`. No retry — a rejected push requires human intervention to correct configuration.
- **Conflict handling**: a GitHub API 422 response from the ref-update call is mapped to `PushResult.conflict(sha)` and recorded on the entity as `CONFLICT`. The conflicting SHA is visible in the UI so the user can rebase manually.
- **No compensation**: every step is either a config probe, an append-only entity write, or a single HTTP call. There is nothing external to roll back beyond what GitHub's own ref-update atomicity provides.
