# PLAN — flight-booking-pipeline

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef tool fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;
  classDef user fill:#0e1a1a,stroke:#22d3ee,color:#22d3ee;

  API[BookingEndpoint]:::ep
  Entity[BookingEntity]:::ese
  WF[BookingPipelineWorkflow]:::wf
  Agent[BookingAgent]:::agent
  Search[SearchTools]:::tool
  Select[SelectTools]:::tool
  Book[BookTools]:::tool
  Guard[BookingGuardrail]:::guard
  View[BookingView]:::view
  App[AppEndpoint]:::ep
  User([User confirms]):::user

  API -->|create| Entity
  API -->|start| WF
  WF -->|searchStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Search
  Agent -->|invokes| Select
  Agent -->|invokes| Book
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|FlightOfferSet / SeatSelection / BookingConfirmation| WF
  WF -->|recordFlights/SeatSelection| Entity
  WF -->|confirmStep: pause| Entity
  User -->|POST /confirm| API
  API -->|confirm| Entity
  Entity -.->|BookingConfirmed unblocks| WF
  WF -->|bookStep runSingleTask| Agent
  WF -->|recordConfirmation| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as BookingEndpoint
  participant E as BookingEntity
  participant W as BookingPipelineWorkflow
  participant A as BookingAgent
  participant G as BookingGuardrail
  participant T as Tools (Search/Select/Book)

  U->>API: POST /api/bookings { origin, destination, travelDate }
  API->>E: create(origin, destination, travelDate)
  E-->>API: { bookingId }
  API->>W: start(bookingId, route)
  W->>E: startSearch
  W->>A: runSingleTask(SEARCH_FLIGHTS, route)
  A->>G: before-tool-call(searchFlights, SEARCH)
  G-->>A: accept (status SEARCHING)
  A->>T: searchFlights + getFareDetails
  T-->>A: List<FlightOffer>
  A-->>W: FlightOfferSet
  W->>E: recordFlights
  W->>A: runSingleTask(SELECT_SEAT, flightOffers)
  A->>G: before-tool-call(getSeatMap, SELECT)
  G-->>A: accept (status SELECTING and flightOffers present)
  A->>T: getSeatMap + rankSeats
  T-->>A: SeatSelection
  A-->>W: SeatSelection
  W->>E: recordSeatSelection
  W->>E: markAwaitingConfirmation
  E-.->>U: SSE event(AWAITING_CONFIRMATION)
  U->>API: POST /api/bookings/{id}/confirm
  API->>E: confirm
  E-.->>W: BookingConfirmed unblocks confirmStep
  W->>E: startBook
  W->>A: runSingleTask(COMMIT_BOOKING, seatSelection)
  A->>G: before-tool-call(commitReservation, BOOK)
  G-->>A: accept (status BOOKING and confirmed and seatSelection present)
  A->>T: commitReservation + getConfirmationDetails
  T-->>A: BookingConfirmation
  A-->>W: BookingConfirmation
  W->>E: recordConfirmation
  E-.->>U: SSE event(COMMITTED)
```

## State machine — `BookingEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> SEARCHING: SearchStarted
  SEARCHING --> SEARCH_DONE: FlightsFound
  SEARCH_DONE --> SELECTING: SelectStarted
  SELECTING --> SEAT_SELECTED: SeatSelected
  SEAT_SELECTED --> AWAITING_CONFIRMATION: AwaitingConfirmation
  AWAITING_CONFIRMATION --> CONFIRMED: BookingConfirmed
  CONFIRMED --> BOOKING: BookStarted
  BOOKING --> COMMITTED: BookingCommitted
  SEARCHING --> FAILED: BookingFailed
  SELECTING --> FAILED: BookingFailed
  BOOKING --> FAILED: BookingFailed
  COMMITTED --> [*]
  FAILED --> [*]
```

`GuardrailRejected` is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  BookingEntity ||--o{ BookingCreated : emits
  BookingEntity ||--o{ SearchStarted : emits
  BookingEntity ||--o{ FlightsFound : emits
  BookingEntity ||--o{ SelectStarted : emits
  BookingEntity ||--o{ SeatSelected : emits
  BookingEntity ||--o{ AwaitingConfirmation : emits
  BookingEntity ||--o{ BookingConfirmed : emits
  BookingEntity ||--o{ BookStarted : emits
  BookingEntity ||--o{ BookingCommitted : emits
  BookingEntity ||--o{ GuardrailRejected : emits
  BookingEntity ||--o{ BookingFailed : emits
  BookingView }o--|| BookingEntity : projects
  BookingPipelineWorkflow }o--|| BookingEntity : reads-and-writes
  BookingAgent ||--o{ FlightOfferSet : returns
  BookingAgent ||--o{ SeatSelection : returns
  BookingAgent ||--o{ BookingConfirmation : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `BookingEndpoint` | `api/BookingEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `BookingEntity` | `application/BookingEntity.java` (state in `domain/BookingRecord.java`, events in `domain/BookingEvent.java`) |
| `BookingPipelineWorkflow` | `application/BookingPipelineWorkflow.java` |
| `BookingAgent` | `application/BookingAgent.java` (tasks in `application/BookingTasks.java`) |
| `SearchTools` | `application/SearchTools.java` |
| `SelectTools` | `application/SelectTools.java` |
| `BookTools` | `application/BookTools.java` |
| `BookingGuardrail` | `application/BookingGuardrail.java` |
| `BookingView` | `application/BookingView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `searchStep` 60 s, `selectStep` 60 s, `confirmStep` 3600 s (human confirmation window), `bookStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(BookingPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4). The 3600 s on `confirmStep` gives a human an hour to review and confirm the itinerary before the workflow fails.
- **Idempotency**: each workflow uses `"pipeline-" + bookingId` as the workflow id; restart of the same bookingId is rejected by the workflow runtime. The agent instance id is `"agent-" + bookingId` so each booking has its own per-task conversation memory.
- **One agent per booking**: `BookingAgent` runs three tasks per booking — SEARCH, SELECT, BOOK — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a misordered or unconfirmed tool call and still let the agent self-correct.
- **Guardrail-driven retry**: when `BookingGuardrail` rejects a tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Confirmation gate is synchronous to the user, asynchronous to the workflow**: `confirmStep` suspends the workflow. The workflow resumes only after the user's `POST /confirm` call writes `BookingConfirmed` onto the entity. No polling loop, no timeout below the 1-hour limit.
- **Task-boundary handoff is the dependency contract**: `searchStep` writes `FlightsFound` BEFORE returning; `selectStep` reads the recorded `FlightOfferSet` from the entity to build its task's instruction context; `bookStep` reads `SeatSelection`. The agent itself is stateless across phases — it never holds search + select + book context in one conversation.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed booking stays at the last successful event; the UI shows the partial state for the user. Reservation rollback is out of scope for this blueprint.
