# PLAN — gemma-food-tour-guide

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

  API[TourEndpoint]:::ep
  Entity[TourRequestEntity]:::ese
  Validator[PreferenceValidator]:::cons
  WF[TourWorkflow]:::wf
  Agent[FoodTourAgent]:::agent
  Guard[ItineraryGuardrail]:::guard
  Scorer[CoverageScorer]:::guard
  View[TourView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|TourRequested| Validator
  Validator -->|attachValidated| Entity
  Validator -->|start workflow| WF
  WF -->|awaitValidatedStep poll| Entity
  WF -->|generateStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|TourItinerary| WF
  WF -->|recordItinerary| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|CoverageResult| WF
  WF -->|recordCoverage| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as TourEndpoint
  participant E as TourRequestEntity
  participant V as PreferenceValidator
  participant W as TourWorkflow
  participant A as FoodTourAgent
  participant G as ItineraryGuardrail
  participant Sc as CoverageScorer

  U->>API: POST /api/tours
  API->>E: submit(preferences)
  E-->>API: { tourId }
  E-.->>V: TourRequested
  V->>V: normalize preferences
  V->>E: attachValidated
  V->>W: start(tourId)
  W->>E: poll getTourRequest
  E-->>W: validated.isPresent()
  W->>E: markGenerating
  W->>A: runSingleTask(preferences + city-profile attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: TourItinerary
  W->>E: recordItinerary(itinerary)
  W->>Sc: score(itinerary, validated)
  Sc-->>W: CoverageResult
  W->>E: recordCoverage(coverage)
  E-.->>U: SSE event(SCORED)
```

## State machine — `TourRequestEntity`

```mermaid
stateDiagram-v2
  [*] --> REQUESTED
  REQUESTED --> PREFERENCES_VALIDATED: PreferencesValidated
  PREFERENCES_VALIDATED --> GENERATING: GenerationStarted
  GENERATING --> ITINERARY_RECORDED: ItineraryRecorded
  ITINERARY_RECORDED --> SCORED: CoverageScored
  GENERATING --> FAILED: TourFailed (agent error)
  REQUESTED --> FAILED: TourFailed (validator error)
  SCORED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  TourRequestEntity ||--o{ TourRequested : emits
  TourRequestEntity ||--o{ PreferencesValidated : emits
  TourRequestEntity ||--o{ GenerationStarted : emits
  TourRequestEntity ||--o{ ItineraryRecorded : emits
  TourRequestEntity ||--o{ CoverageScored : emits
  TourRequestEntity ||--o{ TourFailed : emits
  TourView }o--|| TourRequestEntity : projects
  PreferenceValidator }o--|| TourRequestEntity : subscribes
  TourWorkflow }o--|| TourRequestEntity : reads-and-writes
  FoodTourAgent ||--o{ TourItinerary : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `TourEndpoint` | `api/TourEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `TourRequestEntity` | `application/TourRequestEntity.java` (state in `domain/TourRequest.java`, events in `domain/TourEvent.java`) |
| `PreferenceValidator` | `application/PreferenceValidator.java` |
| `TourWorkflow` | `application/TourWorkflow.java` |
| `FoodTourAgent` | `application/FoodTourAgent.java` (tasks in `application/TourTasks.java`) |
| `ItineraryGuardrail` | `application/ItineraryGuardrail.java` |
| `CoverageScorer` | `application/CoverageScorer.java` |
| `TourView` | `application/TourView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitValidatedStep` 15 s, `generateStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(TourWorkflow::error)`. The 60 s on `generateStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"tour-" + tourId` as the workflow id; the `PreferenceValidator` Consumer is allowed to redeliver `TourRequested` events because `TourRequestEntity.attachValidated` is event-version-guarded — a second normalize attempt against an already-validated request is a no-op.
- **One agent per request**: the AutonomousAgent instance id is `"tour-agent-" + tourId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ItineraryGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `generateStep` fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `CoverageScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same itinerary always scores the same.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
