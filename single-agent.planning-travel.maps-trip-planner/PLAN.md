# PLAN — maps-trip-planner

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

  API[PlanningEndpoint]:::ep
  Entity[TripRequestEntity]:::ese
  Screener[RequestScreener]:::cons
  WF[PlanningWorkflow]:::wf
  Agent[TripPlannerAgent]:::agent
  Guard[ItineraryGuardrail]:::guard
  Scorer[CoverageScorer]:::guard
  Maps[MockMapsClient]:::guard
  View[ItineraryView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|TripRequestSubmitted| Screener
  Screener -->|attachScreen or block| Entity
  Screener -->|start workflow| WF
  WF -->|awaitScreenedStep poll| Entity
  WF -->|planStep runSingleTask| Agent
  Agent -->|geocode/directions/place-details| Maps
  Maps -->|PlaceStop / TransitLeg| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|TripItinerary| WF
  WF -->|recordItinerary| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|CoverageEval| WF
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
  participant API as PlanningEndpoint
  participant E as TripRequestEntity
  participant Sc as RequestScreener
  participant W as PlanningWorkflow
  participant A as TripPlannerAgent
  participant M as MockMapsClient
  participant G as ItineraryGuardrail
  participant Ev as CoverageScorer

  U->>API: POST /api/trips
  API->>E: submit(request)
  E-->>API: { tripId }
  E-.->>Sc: TripRequestSubmitted
  Sc->>Sc: check destinations vs blocked list
  Sc->>E: attachScreen(PASS)
  Sc->>W: start(tripId)
  W->>E: poll getTripPlan
  E-->>W: screen.isPresent() && status != BLOCKED
  W->>E: markPlanning
  W->>A: runSingleTask(trip request instructions)
  A->>M: geocode(destinations)
  M-->>A: PlaceStop[]
  A->>M: directions(stop-to-stop)
  M-->>A: TransitLeg[]
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: TripItinerary
  W->>E: recordItinerary(itinerary)
  W->>Ev: score(itinerary, request)
  Ev-->>W: CoverageEval
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `TripRequestEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SCREENED: RequestScreened (PASS)
  SUBMITTED --> BLOCKED: RequestBlocked
  SCREENED --> PLANNING: PlanningStarted
  PLANNING --> ITINERARY_RECORDED: ItineraryRecorded
  ITINERARY_RECORDED --> EVALUATED: EvaluationScored
  PLANNING --> FAILED: PlanningFailed (agent error)
  SUBMITTED --> FAILED: PlanningFailed (screener error)
  BLOCKED --> [*]
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  TripRequestEntity ||--o{ TripRequestSubmitted : emits
  TripRequestEntity ||--o{ RequestScreened : emits
  TripRequestEntity ||--o{ RequestBlocked : emits
  TripRequestEntity ||--o{ PlanningStarted : emits
  TripRequestEntity ||--o{ ItineraryRecorded : emits
  TripRequestEntity ||--o{ EvaluationScored : emits
  TripRequestEntity ||--o{ PlanningFailed : emits
  ItineraryView }o--|| TripRequestEntity : projects
  RequestScreener }o--|| TripRequestEntity : subscribes
  PlanningWorkflow }o--|| TripRequestEntity : reads-and-writes
  TripPlannerAgent ||--o{ TripItinerary : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PlanningEndpoint` | `api/PlanningEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `TripRequestEntity` | `application/TripRequestEntity.java` (state in `domain/TripPlan.java`, events in `domain/TripEvent.java`) |
| `RequestScreener` | `application/RequestScreener.java` |
| `PlanningWorkflow` | `application/PlanningWorkflow.java` |
| `TripPlannerAgent` | `application/TripPlannerAgent.java` (tasks in `application/TripTasks.java`) |
| `ItineraryGuardrail` | `application/ItineraryGuardrail.java` |
| `CoverageScorer` | `application/CoverageScorer.java` |
| `MockMapsClient` | `application/MockMapsClient.java` |
| `ItineraryView` | `application/ItineraryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitScreenedStep` 15 s, `planStep` 90 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(PlanningWorkflow::error)`. The 90 s on `planStep` accommodates LLM latency plus simulated Maps tool round-trips (Lesson 4).
- **Blocked path**: when `RequestScreener` calls `TripRequestEntity.block(reason)`, the entity emits `RequestBlocked` and transitions to BLOCKED. The `awaitScreenedStep` in the workflow detects `status == BLOCKED` and transitions the workflow to its terminal no-op path; no agent call is made.
- **Idempotency**: every workflow uses `"plan-" + tripId` as the workflow id; the `RequestScreener` Consumer is allowed to redeliver `TripRequestSubmitted` events because `TripRequestEntity.attachScreen` is event-version-guarded — a second screen attempt against an already-screened trip is a no-op.
- **One agent per trip**: the AutonomousAgent instance id is `"planner-" + tripId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ItineraryGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `planStep` fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `CoverageScorer` runs in-process inside `evalStep`. No LLM call, no external service.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call with in-process tool stubs.
