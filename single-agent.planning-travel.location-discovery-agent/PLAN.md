# PLAN — location-discovery-agent

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

  API[SearchEndpoint]:::ep
  Entity[SearchEntity]:::ese
  Sanitizer[CoordinateSanitizer]:::cons
  WF[DiscoveryWorkflow]:::wf
  Agent[LocationDiscoveryAgent]:::agent
  Guard[RecommendationGuardrail]:::guard
  Scorer[EvaluationScorer]:::guard
  View[SearchView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|SearchSubmitted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|discoverStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|PlaceRecommendation| WF
  WF -->|recordRecommendation| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|EvalResult| WF
  WF -->|recordEvaluation| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as SearchEndpoint
  participant E as SearchEntity
  participant S as CoordinateSanitizer
  participant W as DiscoveryWorkflow
  participant A as LocationDiscoveryAgent
  participant G as RecommendationGuardrail
  participant Sc as EvaluationScorer

  U->>API: POST /api/searches
  API->>E: submit(request)
  E-->>API: { searchId }
  E-.->>S: SearchSubmitted
  S->>S: redact coordinates → bbox token
  S->>S: load candidate places within radius
  S->>E: attachSanitized
  S->>W: start(searchId)
  W->>E: poll getSearch
  E-->>W: sanitized.isPresent()
  W->>E: markDiscovering
  W->>A: runSingleTask(query + places.json attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: PlaceRecommendation
  W->>E: recordRecommendation(recommendation)
  W->>Sc: score(recommendation, request)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `SearchEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> COORDINATE_SANITIZED: CoordinateSanitized
  COORDINATE_SANITIZED --> DISCOVERING: DiscoveryStarted
  DISCOVERING --> RECOMMENDATION_RECORDED: RecommendationRecorded
  RECOMMENDATION_RECORDED --> EVALUATED: EvaluationScored
  DISCOVERING --> FAILED: SearchFailed (agent error)
  SUBMITTED --> FAILED: SearchFailed (sanitizer error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  SearchEntity ||--o{ SearchSubmitted : emits
  SearchEntity ||--o{ CoordinateSanitized : emits
  SearchEntity ||--o{ DiscoveryStarted : emits
  SearchEntity ||--o{ RecommendationRecorded : emits
  SearchEntity ||--o{ EvaluationScored : emits
  SearchEntity ||--o{ SearchFailed : emits
  SearchView }o--|| SearchEntity : projects
  CoordinateSanitizer }o--|| SearchEntity : subscribes
  DiscoveryWorkflow }o--|| SearchEntity : reads-and-writes
  LocationDiscoveryAgent ||--o{ PlaceRecommendation : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SearchEndpoint` | `api/SearchEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SearchEntity` | `application/SearchEntity.java` (state in `domain/Search.java`, events in `domain/SearchEvent.java`) |
| `CoordinateSanitizer` | `application/CoordinateSanitizer.java` |
| `DiscoveryWorkflow` | `application/DiscoveryWorkflow.java` |
| `LocationDiscoveryAgent` | `application/LocationDiscoveryAgent.java` (tasks in `application/SearchTasks.java`) |
| `RecommendationGuardrail` | `application/RecommendationGuardrail.java` |
| `EvaluationScorer` | `application/EvaluationScorer.java` |
| `SearchView` | `application/SearchView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `discoverStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(DiscoveryWorkflow::error)`. The 60 s on `discoverStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"discovery-" + searchId` as the workflow id; the `CoordinateSanitizer` Consumer is allowed to redeliver `SearchSubmitted` events because `SearchEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized search is a no-op.
- **One agent per search**: the AutonomousAgent instance id is `"discoverer-" + searchId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `RecommendationGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `discoverStep` fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `EvaluationScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same recommendation always scores the same. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
