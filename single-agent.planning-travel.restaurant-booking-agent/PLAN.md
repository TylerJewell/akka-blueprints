# PLAN — restaurant-booking-agent

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
  classDef stub fill:#0e1e0e,stroke:#6ee7b7,color:#6ee7b7;

  API[BookingEndpoint]:::ep
  Entity[BookingEntity]:::ese
  Consumer[ConfirmationConsumer]:::cons
  WF[BookingWorkflow]:::wf
  Agent[BookingAgent]:::agent
  Guard[ReservationGuardrail]:::guard
  Stub[BookingStub]:::stub
  View[BookingView]:::view
  App[AppEndpoint]:::ep

  API -->|initiate + start workflow| Entity
  API -->|confirm/decline signal| Consumer
  Entity -.->|ConfirmationRequested| Consumer
  Consumer -->|receiveConfirmation| WF
  WF -->|collectStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -->|pass/reject| Agent
  Agent -->|BookingProposal| WF
  WF -->|requestConfirmation| Entity
  WF -->|confirmStep wait| Consumer
  WF -->|commitStep createBooking| Stub
  Stub -->|BookingConfirmation| WF
  WF -->|complete/fail| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as BookingEndpoint
  participant E as BookingEntity
  participant W as BookingWorkflow
  participant A as BookingAgent
  participant G as ReservationGuardrail
  participant S as BookingStub
  participant C as ConfirmationConsumer

  U->>API: POST /api/bookings
  API->>E: initiate(request)
  API->>W: start(bookingId)
  E-->>API: { bookingId }
  W->>E: startCollection
  W->>A: runSingleTask(request + catalogue)
  A->>G: before-tool-call(proposal params)
  G-->>A: accept
  A-->>W: BookingProposal
  W->>E: requestConfirmation(proposal)
  E-.->>C: ConfirmationRequested
  E-.->>U: SSE event(PENDING_CONFIRMATION)
  U->>API: POST /api/bookings/{id}/confirm
  API->>C: signal(CONFIRMED)
  C->>W: receiveConfirmation(CONFIRMED)
  W->>E: startCommit
  W->>S: createBooking(proposal)
  S-->>W: BookingConfirmation
  W->>E: complete(confirmation)
  E-.->>U: SSE event(CONFIRMED)
```

## State machine — `BookingEntity`

```mermaid
stateDiagram-v2
  [*] --> INITIATED
  INITIATED --> COLLECTING: CollectionStarted
  COLLECTING --> PENDING_CONFIRMATION: ConfirmationRequested
  PENDING_CONFIRMATION --> COMMITTING: BookingConfirmed (user confirm)
  PENDING_CONFIRMATION --> DECLINED: BookingDeclined (user decline)
  COMMITTING --> CONFIRMED: BookingCompleted
  COMMITTING --> FAILED: BookingFailed (stub error)
  COLLECTING --> FAILED: BookingFailed (agent error)
  CONFIRMED --> [*]
  DECLINED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  BookingEntity ||--o{ BookingInitiated : emits
  BookingEntity ||--o{ CollectionStarted : emits
  BookingEntity ||--o{ ConfirmationRequested : emits
  BookingEntity ||--o{ BookingConfirmed : emits
  BookingEntity ||--o{ BookingDeclined : emits
  BookingEntity ||--o{ CommitStarted : emits
  BookingEntity ||--o{ BookingCompleted : emits
  BookingEntity ||--o{ BookingFailed : emits
  BookingView }o--|| BookingEntity : projects
  ConfirmationConsumer }o--|| BookingEntity : subscribes
  BookingWorkflow }o--|| BookingEntity : reads-and-writes
  BookingAgent ||--o{ BookingProposal : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `BookingEndpoint` | `api/BookingEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `BookingEntity` | `application/BookingEntity.java` (state in `domain/Booking.java`, events in `domain/BookingEvent.java`) |
| `ConfirmationConsumer` | `application/ConfirmationConsumer.java` |
| `BookingWorkflow` | `application/BookingWorkflow.java` |
| `BookingAgent` | `application/BookingAgent.java` (tasks in `application/BookingTasks.java`) |
| `ReservationGuardrail` | `application/ReservationGuardrail.java` |
| `BookingStub` | `application/BookingStub.java` |
| `BookingView` | `application/BookingView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `collectStep` 60 s, `confirmStep` 300 s, `commitStep` 15 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(BookingWorkflow::error)`. The 300 s on `confirmStep` gives the user a 5-minute window to confirm or decline before the workflow times out and fails.
- **Idempotency**: every workflow uses `"booking-" + bookingId` as the workflow id. `BookingEntity.initiate` is event-version-guarded — a duplicate POST to `/api/bookings` for the same bookingId is a no-op.
- **One agent per booking**: the AutonomousAgent instance id is `"booker-" + bookingId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(4)` caps guardrail-triggered retries at 4.
- **Guardrail-driven retry**: when `ReservationGuardrail` rejects a tool-call candidate, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow's `collectStep` fails over to `error` and the entity transitions to `FAILED`.
- **Confirmation window**: `confirmStep` holds the workflow open for up to 300 s. If neither confirm nor decline arrives, the step times out and the workflow transitions to `error`, which calls `BookingEntity.fail("confirmation-timeout")`.
- **No saga / no compensation**: `commitStep` either returns a `BookingConfirmation` or fails. There is no external resource to roll back — the booking stub is the only write.
