# PLAN — airline-cs

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
  classDef svc fill:#0e1a2a,stroke:#60a5fa,color:#60a5fa;

  API[ServiceEndpoint]:::ep
  Entity[RequestEntity]:::ese
  Sanitizer[MessageSanitizer]:::cons
  WF[ServiceWorkflow]:::wf
  Agent[CustomerServiceAgent]:::agent
  Guard[ModificationGuardrail]:::guard
  Booking[BookingService]:::svc
  View[RequestView]:::view
  App[AppEndpoint]:::ep

  API -->|receive| Entity
  Entity -.->|RequestReceived| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|serveStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|searchBooking / fileComplaint| Booking
  Agent -->|modifyReservation / cancelReservation| Booking
  Agent -.->|ConfirmationRequested| WF
  WF -->|hitlStep pause| Entity
  API -->|POST confirmation| WF
  WF -->|ConfirmationReceived resume| Agent
  Agent -->|ServiceOutcome| WF
  WF -->|complete| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (seat change with HITL)

```mermaid
sequenceDiagram
  autonumber
  participant U as Customer (UI)
  participant API as ServiceEndpoint
  participant E as RequestEntity
  participant S as MessageSanitizer
  participant W as ServiceWorkflow
  participant A as CustomerServiceAgent
  participant G as ModificationGuardrail
  participant B as BookingService

  U->>API: POST /api/requests
  API->>E: receive(request)
  E-->>API: { requestId }
  E-.->>S: RequestReceived
  S->>S: redact PII
  S->>E: attachSanitized
  S->>W: start(requestId)
  W->>E: poll getRequest
  E-->>W: sanitized.isPresent()
  W->>E: markProcessing
  W->>A: runSingleTask(sanitizedMessage + bookingContext)
  A->>B: searchBooking(ABC123)
  B-->>A: BookingRecord
  A->>G: before-tool-call(modifyReservation, args)
  G-->>A: MISSING_CONFIRMATION_TOKEN — reject
  A->>A: call requestConfirmation(proposedChange)
  A-.->>W: ConfirmationRequested signal
  W->>E: requestConfirmation(confirmationRequest)
  E-.->>U: SSE event(AWAITING_CONFIRMATION)
  U->>API: POST /api/requests/{id}/confirmation {action:"confirm"}
  API->>W: resume(token)
  W->>E: receiveConfirmation(token)
  W-->>A: resume with token
  A->>G: before-tool-call(modifyReservation, args+token)
  G-->>A: accept
  A->>B: modifyReservation(ABC123, change, token)
  B-->>A: ModificationResult
  A-->>W: ServiceOutcome{RESOLVED}
  W->>E: complete(outcome)
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `RequestEntity`

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: MessageSanitized
  SANITIZED --> PROCESSING: ProcessingStarted
  PROCESSING --> AWAITING_CONFIRMATION: ConfirmationRequested
  AWAITING_CONFIRMATION --> PROCESSING: ConfirmationReceived
  AWAITING_CONFIRMATION --> CANCELLED: RequestCancelled
  PROCESSING --> COMPLETED: RequestCompleted
  PROCESSING --> FAILED: RequestFailed (agent/tool error)
  RECEIVED --> FAILED: RequestFailed (sanitizer error)
  COMPLETED --> [*]
  CANCELLED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  RequestEntity ||--o{ RequestReceived : emits
  RequestEntity ||--o{ MessageSanitized : emits
  RequestEntity ||--o{ ProcessingStarted : emits
  RequestEntity ||--o{ ConfirmationRequested : emits
  RequestEntity ||--o{ ConfirmationReceived : emits
  RequestEntity ||--o{ RequestCompleted : emits
  RequestEntity ||--o{ RequestCancelled : emits
  RequestEntity ||--o{ RequestFailed : emits
  RequestView }o--|| RequestEntity : projects
  MessageSanitizer }o--|| RequestEntity : subscribes
  ServiceWorkflow }o--|| RequestEntity : reads-and-writes
  CustomerServiceAgent ||--o{ ServiceOutcome : returns
  ServiceWorkflow }o--|| CustomerServiceAgent : invokes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ServiceEndpoint` | `api/ServiceEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `RequestEntity` | `application/RequestEntity.java` (state in `domain/ServiceRequestState.java`, events in `domain/RequestEvent.java`) |
| `MessageSanitizer` | `application/MessageSanitizer.java` |
| `ServiceWorkflow` | `application/ServiceWorkflow.java` |
| `CustomerServiceAgent` | `application/CustomerServiceAgent.java` (tasks in `application/ServiceTasks.java`) |
| `ModificationGuardrail` | `application/ModificationGuardrail.java` |
| `BookingService` | `application/BookingService.java` |
| `RequestView` | `application/RequestView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `serveStep` 120 s, `hitlStep` 3600 s (customer paced), `completeStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ServiceWorkflow::error)`. The 120 s on `serveStep` accommodates multi-turn ReAct loops with LLM latency plus potential mid-loop HITL round-trips (Lesson 4).
- **HITL suspension**: when the agent calls `requestConfirmation`, the workflow transitions to `hitlStep` and suspends. The entity records `ConfirmationRequested`. A call to `POST /api/requests/{id}/confirmation` resumes the workflow. The 3600 s hitlStep timeout is intentional — customers may take minutes to respond. If the timeout expires, the step fails over to `error`.
- **Idempotency**: every workflow uses `"service-" + requestId` as the workflow id; `MessageSanitizer` is allowed to redeliver `RequestReceived` because `RequestEntity.attachSanitized` is event-version-guarded.
- **One agent per request**: agent instance id is `"agent-" + requestId`, giving each task its own conversation context. `maxIterationsPerTask(8)` accommodates multi-tool ReAct loops including one confirmation round-trip.
- **Guardrail-driven re-route**: when `ModificationGuardrail` rejects a tool call, the rejection returns a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`. If the agent exhausts its budget without obtaining a confirmation token, the workflow fails over to `error`.
- **BookingService is not an agent**: it is a plain Java class, not an AutonomousAgent. Tool call routing stays inside the agent's ReAct loop. This is what makes the single-agent invariant hold.
