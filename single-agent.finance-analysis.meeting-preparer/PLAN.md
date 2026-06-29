# PLAN — meeting-preparer

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
  classDef eval fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[BriefingEndpoint]:::ep
  Entity[BriefingEntity]:::ese
  Sanitizer[ContactSanitizer]:::cons
  WF[BriefingWorkflow]:::wf
  Agent[BriefingAgent]:::agent
  Evaluator[BriefingEvaluator]:::eval
  View[BriefingView]:::view
  App[AppEndpoint]:::ep

  API -->|request| Entity
  Entity -.->|MeetingRequested| Sanitizer
  Sanitizer -->|attachSanitizedData| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|briefStep runSingleTask| Agent
  Agent -->|MeetingBrief| WF
  WF -->|recordBrief| Entity
  WF -->|evalStep score| Evaluator
  Evaluator -->|BriefEval| WF
  WF -->|recordEval| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as BriefingEndpoint
  participant E as BriefingEntity
  participant S as ContactSanitizer
  participant W as BriefingWorkflow
  participant A as BriefingAgent
  participant Ev as BriefingEvaluator

  U->>API: POST /api/briefs
  API->>E: request(meetingRequest)
  E-->>API: { briefId }
  E-.->>S: MeetingRequested
  S->>S: redact CRM PII
  S->>E: attachSanitizedData
  S->>W: start(briefId)
  W->>E: poll getBriefing
  E-->>W: sanitized.isPresent()
  W->>E: markBriefing
  W->>A: runSingleTask(meeting context + attachments)
  A-->>W: MeetingBrief
  W->>E: recordBrief(brief)
  W->>Ev: score(brief)
  Ev-->>W: BriefEval
  W->>E: recordEval(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `BriefingEntity`

```mermaid
stateDiagram-v2
  [*] --> REQUESTED
  REQUESTED --> DATA_SANITIZED: ContactDataSanitized
  DATA_SANITIZED --> BRIEFING: BriefingStarted
  BRIEFING --> BRIEF_READY: BriefReady
  BRIEF_READY --> EVALUATED: BriefEvaluated
  BRIEFING --> FAILED: BriefingFailed (agent error)
  REQUESTED --> FAILED: BriefingFailed (sanitizer error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  BriefingEntity ||--o{ MeetingRequested : emits
  BriefingEntity ||--o{ ContactDataSanitized : emits
  BriefingEntity ||--o{ BriefingStarted : emits
  BriefingEntity ||--o{ BriefReady : emits
  BriefingEntity ||--o{ BriefEvaluated : emits
  BriefingEntity ||--o{ BriefingFailed : emits
  BriefingView }o--|| BriefingEntity : projects
  ContactSanitizer }o--|| BriefingEntity : subscribes
  BriefingWorkflow }o--|| BriefingEntity : reads-and-writes
  BriefingAgent ||--o{ MeetingBrief : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `BriefingEndpoint` | `api/BriefingEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `BriefingEntity` | `application/BriefingEntity.java` (state in `domain/Briefing.java`, events in `domain/BriefingEvent.java`) |
| `ContactSanitizer` | `application/ContactSanitizer.java` |
| `BriefingWorkflow` | `application/BriefingWorkflow.java` |
| `BriefingAgent` | `application/BriefingAgent.java` (tasks in `application/BriefingTasks.java`) |
| `BriefingEvaluator` | `application/BriefingEvaluator.java` |
| `BriefingView` | `application/BriefingView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `briefStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(BriefingWorkflow::error)`. The 60 s on `briefStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"briefing-" + briefId` as the workflow id; the `ContactSanitizer` Consumer is allowed to redeliver `MeetingRequested` events because `BriefingEntity.attachSanitizedData` is event-version-guarded — a second sanitize attempt against an already-sanitized briefing is a no-op.
- **One agent per brief**: the AutonomousAgent instance id is `"briefer-" + briefId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps retries at 3.
- **Eval is synchronous and deterministic**: `BriefingEvaluator` runs in-process inside `evalStep`. No LLM call, no external service — the same brief always scores the same.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
- **Three attachments**: `briefStep` passes `sanitized-crm.txt`, `financial-highlights.txt`, and `news-items.txt` as `TaskDef.attachment(...)` — never inlined into the instruction string.
