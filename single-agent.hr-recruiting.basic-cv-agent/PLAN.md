# PLAN — basic-cv-agent

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

  API[CvEndpoint]:::ep
  Entity[CvEntity]:::ese
  Sanitizer[ProfileSanitizer]:::cons
  WF[CvWorkflow]:::wf
  Agent[CvGeneratorAgent]:::agent
  View[CvView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|ProfileSubmitted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|generateStep runSingleTask| Agent
  Agent -->|GeneratedCv| WF
  WF -->|recordGenerated| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as CvEndpoint
  participant E as CvEntity
  participant S as ProfileSanitizer
  participant W as CvWorkflow
  participant A as CvGeneratorAgent

  U->>API: POST /api/cv-requests
  API->>E: submit(request)
  E-->>API: { requestId }
  E-.->>S: ProfileSubmitted
  S->>S: redact PII
  S->>E: attachSanitized
  S->>W: start(requestId)
  W->>E: poll getRequest
  E-->>W: sanitized.isPresent()
  W->>E: markGenerating
  W->>A: runSingleTask(instructions + attachment)
  A-->>W: GeneratedCv
  W->>W: validate headline+sections non-empty
  W->>E: recordGenerated(generatedCv)
  E-.->>U: SSE event(CV_GENERATED)
```

## State machine — `CvEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SANITIZED: ProfileSanitized
  SANITIZED --> GENERATING: GenerationStarted
  GENERATING --> CV_GENERATED: CvGenerated
  GENERATING --> FAILED: GenerationFailed (agent error or empty output)
  SUBMITTED --> FAILED: GenerationFailed (sanitizer error)
  CV_GENERATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  CvEntity ||--o{ ProfileSubmitted : emits
  CvEntity ||--o{ ProfileSanitized : emits
  CvEntity ||--o{ GenerationStarted : emits
  CvEntity ||--o{ CvGenerated : emits
  CvEntity ||--o{ GenerationFailed : emits
  CvView }o--|| CvEntity : projects
  ProfileSanitizer }o--|| CvEntity : subscribes
  CvWorkflow }o--|| CvEntity : reads-and-writes
  CvGeneratorAgent ||--o{ GeneratedCv : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `CvEndpoint` | `api/CvEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `CvEntity` | `application/CvEntity.java` (state in `domain/CvRequestState.java`, events in `domain/CvEvent.java`) |
| `ProfileSanitizer` | `application/ProfileSanitizer.java` |
| `CvWorkflow` | `application/CvWorkflow.java` |
| `CvGeneratorAgent` | `application/CvGeneratorAgent.java` (tasks in `application/CvTasks.java`) |
| `CvView` | `application/CvView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `generateStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(CvWorkflow::error)`. The 60 s on `generateStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"cv-" + requestId` as the workflow id; the `ProfileSanitizer` Consumer is allowed to redeliver `ProfileSubmitted` events because `CvEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized request is a no-op.
- **One agent per request**: the AutonomousAgent instance id is `"cv-gen-" + requestId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps retries at 3.
- **No guardrail on the agent**: structural validation of the returned `GeneratedCv` (headline non-empty, sections list non-empty) is done by `generateStep` in the workflow before calling `CvEntity.recordGenerated`. A failed validation transitions the entity to `FAILED` via `CvEntity.fail(reason)`. This is intentional — the blueprint keeps the single control (S1) minimal so the pattern is readable.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
