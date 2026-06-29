# PLAN — personalized-shopper

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

  API[ShoppingEndpoint]:::ep
  Entity[ShoppingSessionEntity]:::ese
  Sanitizer[ProfileSanitizer]:::cons
  WF[RecommendationWorkflow]:::wf
  Agent[ShoppingAdvisorAgent]:::agent
  Scorer[FreshnessScorer]:::scorer
  View[RecommendationView]:::view
  App[AppEndpoint]:::ep

  API -->|start| Entity
  Entity -.->|SessionStarted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|recommendStep runSingleTask| Agent
  Agent -->|RecommendationSet| WF
  WF -->|recordRecommendations| Entity
  WF -->|freshnessStep score| Scorer
  Scorer -->|FreshnessResult| WF
  WF -->|recordFreshness| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ShoppingEndpoint
  participant E as ShoppingSessionEntity
  participant S as ProfileSanitizer
  participant W as RecommendationWorkflow
  participant A as ShoppingAdvisorAgent
  participant Sc as FreshnessScorer

  U->>API: POST /api/sessions
  API->>E: start(profile)
  E-->>API: { sessionId }
  E-.->>S: SessionStarted
  S->>S: strip PII from profile
  S->>E: attachSanitized
  S->>W: start(sessionId)
  W->>E: poll getSession
  E-->>W: sanitized.isPresent()
  W->>E: markRecommending
  W->>A: runSingleTask(catalog + profile attachment)
  A-->>W: RecommendationSet
  W->>E: recordRecommendations(set)
  W->>Sc: score(set, catalog)
  Sc-->>W: FreshnessResult
  W->>E: recordFreshness(freshness)
  E-.->>U: SSE event(FRESHNESS_SCORED)
```

## State machine — `ShoppingSessionEntity`

```mermaid
stateDiagram-v2
  [*] --> STARTED
  STARTED --> PROFILE_SANITIZED: ProfileSanitized
  PROFILE_SANITIZED --> RECOMMENDING: RecommendationStarted
  RECOMMENDING --> RECOMMENDATIONS_READY: RecommendationsRecorded
  RECOMMENDATIONS_READY --> FRESHNESS_SCORED: FreshnessScored
  RECOMMENDING --> FAILED: SessionFailed (agent error)
  STARTED --> FAILED: SessionFailed (sanitizer error)
  FRESHNESS_SCORED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ShoppingSessionEntity ||--o{ SessionStarted : emits
  ShoppingSessionEntity ||--o{ ProfileSanitized : emits
  ShoppingSessionEntity ||--o{ RecommendationStarted : emits
  ShoppingSessionEntity ||--o{ RecommendationsRecorded : emits
  ShoppingSessionEntity ||--o{ FreshnessScored : emits
  ShoppingSessionEntity ||--o{ SessionFailed : emits
  RecommendationView }o--|| ShoppingSessionEntity : projects
  ProfileSanitizer }o--|| ShoppingSessionEntity : subscribes
  RecommendationWorkflow }o--|| ShoppingSessionEntity : reads-and-writes
  ShoppingAdvisorAgent ||--o{ RecommendationSet : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ShoppingEndpoint` | `api/ShoppingEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ShoppingSessionEntity` | `application/ShoppingSessionEntity.java` (state in `domain/ShoppingSession.java`, events in `domain/SessionEvent.java`) |
| `ProfileSanitizer` | `application/ProfileSanitizer.java` |
| `RecommendationWorkflow` | `application/RecommendationWorkflow.java` |
| `ShoppingAdvisorAgent` | `application/ShoppingAdvisorAgent.java` (tasks in `application/ShoppingTasks.java`) |
| `FreshnessScorer` | `application/FreshnessScorer.java` |
| `RecommendationView` | `application/RecommendationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `recommendStep` 60 s, `freshnessStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(RecommendationWorkflow::error)`. The 60 s on `recommendStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"rec-" + sessionId` as the workflow id; the `ProfileSanitizer` Consumer is allowed to redeliver `SessionStarted` events because `ShoppingSessionEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized session is a no-op.
- **One agent per session**: the AutonomousAgent instance id is `"advisor-" + sessionId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps retries at 3.
- **Freshness is synchronous and deterministic**: `FreshnessScorer` runs in-process inside `freshnessStep`. No LLM call, no external service — the same recommendation set against the same catalog always scores the same. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
- **Catalog is passed as instructions**: the full catalog snapshot is formatted into the agent's `TaskDef.instructions(...)` text so it is bounded by the model's context window and does not require an attachment slot. The sanitized profile — not the catalog — occupies the attachment.
