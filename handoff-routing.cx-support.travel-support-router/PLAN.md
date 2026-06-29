# PLAN — travel-support-router

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'fontFamily':'Instrument Sans, system-ui, sans-serif'
}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef autonomous fill:#0e2a1e,stroke:#3fb950,color:#3fb950;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Sim[RequestSimulator]:::ta
  Queue[RequestQueue]:::ese
  Sanit[PiiSanitizer]:::cons
  Router[RouterAgent]:::agent
  Flight[FlightSpecialist]:::autonomous
  Hotel[HotelSpecialist]:::autonomous
  Car[CarRentalSpecialist]:::autonomous
  Excur[ExcursionSpecialist]:::autonomous
  Judge[RoutingJudge]:::agent
  Guard[BookingGuardrail]:::agent
  Confirm[ConfirmationGate]:::agent
  WF[TravelWorkflow]:::wf
  Entity[RequestEntity]:::ese
  View[RequestView]:::view
  Scorer[RouterEvalScorer]:::cons
  API[TravelEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| Sanit
  Sanit -->|register + sanitize| Entity
  Sanit -->|start workflow| WF
  WF -->|route| Router
  WF -->|RESOLVE task| Flight
  WF -->|RESOLVE task| Hotel
  WF -->|RESOLVE task| Car
  WF -->|RESOLVE task| Excur
  WF -->|check tool call| Guard
  WF -->|confirm change| Confirm
  WF -->|emit events| Entity
  Entity -.->|projects| View
  Entity -.->|subscribes| Scorer
  Scorer -->|score| Judge
  Scorer -->|recordRoutingScore| Entity
  API -->|unblock / submit / confirm| Entity
  API -->|query / SSE| View
  App -->|static UI + metadata| API
```

Solid arrows = synchronous component calls. Dashed arrows = event subscriptions and scheduler ticks.

## Interaction sequence — J1 (flight rebooking happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'actorTextColor':'#ffffff','noteTextColor':'#ffffff','sequenceNumberColor':'#000'
}}}%%
sequenceDiagram
  autonumber
  participant Sim as RequestSimulator
  participant Q as RequestQueue
  participant S as PiiSanitizer
  participant E as RequestEntity
  participant W as TravelWorkflow
  participant R as RouterAgent
  participant F as FlightSpecialist
  participant G as BookingGuardrail
  participant C as ConfirmationGate
  participant Sc as RouterEvalScorer
  participant J as RoutingJudge

  Sim->>Q: receive(IncomingRequest)
  Q->>S: InboundRequestReceived
  S->>E: registerIncoming + attachSanitized
  S->>W: start(requestId, sanitized)
  W->>R: route(sanitized)
  R-->>W: RoutingDecision{FLIGHTS}
  W->>E: recordRouting(decision) [emits RoutingDecided]
  E->>Sc: RoutingDecided event
  Sc->>J: score(sanitized, decision)
  J-->>Sc: RoutingScore
  Sc->>E: recordRoutingScore [emits RoutingScored]
  W->>E: recordRoutedCategory(FLIGHTS) [emits RequestRouted]
  W->>F: runSingleTask(RESOLVE, prompt)
  F-->>W: Resolution (proposed change)
  W->>G: check(sanitized, proposedAction)
  G-->>W: GuardrailVerdict{allowed=true}
  W->>E: recordGuardrailVerdict [emits GuardrailVerdictAttached]
  W->>C: summarize(sanitized, resolution)
  C-->>W: ConfirmationRequest{summary, deadline}
  W->>E: recordConfirmationRequest [emits ConfirmationRequested] — PAUSED
  Note over W: workflow pauses; passenger receives confirmation prompt
  W->>E: recordConfirmationOutcome(CONFIRM) [emits PassengerConfirmed]
  W->>E: recordDraft(resolution) [emits ResolutionDrafted]
  W->>E: publish(resolution) [emits ResolutionPublished, status RESOLVED]
```

The eval-event sequence (steps 7–10) runs concurrently with the workflow's continuation — `RouterEvalScorer` is a Consumer reading the entity's event stream, independent of `TravelWorkflow`. Both writes target the same `RequestEntity`; the entity's commands are idempotent on `requestId`.

## State machine — `RequestEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: RequestSanitized
  SANITIZED --> ROUTED: RoutingDecided
  ROUTED --> ROUTED_FLIGHTS: category = FLIGHTS
  ROUTED --> ROUTED_HOTELS: category = HOTELS
  ROUTED --> ROUTED_CAR_RENTAL: category = CAR_RENTAL
  ROUTED --> ROUTED_EXCURSIONS: category = EXCURSIONS
  ROUTED --> ESCALATED: category = UNCLEAR
  ROUTED_FLIGHTS --> GUARDRAIL_CLEARED: guardrail.allowed
  ROUTED_HOTELS --> GUARDRAIL_CLEARED: guardrail.allowed
  ROUTED_CAR_RENTAL --> GUARDRAIL_CLEARED: guardrail.allowed
  ROUTED_EXCURSIONS --> GUARDRAIL_CLEARED: guardrail.allowed
  ROUTED_FLIGHTS --> BLOCKED: guardrail.rejected
  ROUTED_HOTELS --> BLOCKED: guardrail.rejected
  ROUTED_CAR_RENTAL --> BLOCKED: guardrail.rejected
  ROUTED_EXCURSIONS --> BLOCKED: guardrail.rejected
  GUARDRAIL_CLEARED --> AWAITING_CONFIRMATION: ConfirmationRequested
  AWAITING_CONFIRMATION --> RESOLUTION_DRAFTED: PassengerConfirmed
  AWAITING_CONFIRMATION --> PASSENGER_REJECTED: PassengerRejected
  RESOLUTION_DRAFTED --> RESOLVED: ResolutionPublished
  BLOCKED --> RESOLVED: operator unblock
  RESOLVED --> [*]
  PASSENGER_REJECTED --> [*]
  ESCALATED --> [*]
```

The `RoutingScored` event does not change `status`; it attaches the eval result. The state machine treats it as a no-op transition and omits it from the diagram.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  RequestEntity ||--o{ RequestRegistered : emits
  RequestEntity ||--o{ RequestSanitized : emits
  RequestEntity ||--o{ RoutingDecided : emits
  RequestEntity ||--o{ RequestRouted : emits
  RequestEntity ||--o{ GuardrailVerdictAttached : emits
  RequestEntity ||--o{ ConfirmationRequested : emits
  RequestEntity ||--o{ PassengerConfirmed : emits
  RequestEntity ||--o{ PassengerRejected : emits
  RequestEntity ||--o{ ResolutionDrafted : emits
  RequestEntity ||--o{ ResolutionPublished : emits
  RequestEntity ||--o{ ResolutionBlocked : emits
  RequestEntity ||--o{ RequestEscalated : emits
  RequestEntity ||--o{ RoutingScored : emits
  RequestView }o--|| RequestEntity : projects
  RequestQueue ||--o{ InboundRequestReceived : emits
  PiiSanitizer }o--|| RequestQueue : subscribes
  RouterEvalScorer }o--|| RequestEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RequestSimulator` | `application/RequestSimulator.java` |
| `RequestQueue` | `application/RequestQueue.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `RouterAgent` | `application/RouterAgent.java` |
| `FlightSpecialist` | `application/FlightSpecialist.java` |
| `HotelSpecialist` | `application/HotelSpecialist.java` |
| `CarRentalSpecialist` | `application/CarRentalSpecialist.java` |
| `ExcursionSpecialist` | `application/ExcursionSpecialist.java` |
| `RoutingJudge` | `application/RoutingJudge.java` |
| `BookingGuardrail` | `application/BookingGuardrail.java` |
| `ConfirmationGate` | `application/ConfirmationGate.java` |
| `TravelWorkflow` | `application/TravelWorkflow.java` |
| `RequestEntity` | `application/RequestEntity.java` (state in `domain/TravelRequest.java`, events in `domain/RequestEvent.java`) |
| `RequestView` | `application/RequestView.java` |
| `RouterEvalScorer` | `application/RouterEvalScorer.java` |
| `TravelEndpoint` | `api/TravelEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/TravelTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `routeStep` 20 s, `guardrailStep` 20 s, `specialistStep` / `confirmStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the request to `ESCALATED` with the failure reason captured.
- **Idempotency.** Every per-request primitive is keyed by `requestId`: `RequestEntity` id is `requestId`; `TravelWorkflow` id is `requestId`; agent sessions for `RouterAgent`, `RoutingJudge`, `BookingGuardrail`, and `ConfirmationGate` use `requestId`. Duplicate sanitize events fold into a single workflow start.
- **Race between eval and workflow.** `RouterEvalScorer` (Consumer) and `TravelWorkflow` both append events to the same `RequestEntity`. Order is not guaranteed but does not matter: `RoutingScored` only mutates `routingScore`, never `status`.
- **HITL pause duration.** The `confirmStep` has a 60-second timeout, but for demo purposes the mock responder auto-confirms after a few seconds so that the simulator's 30-second cycle can complete end-to-end. In a deployed setting the timeout would be configured to the passenger's SLA.
- **No saga compensation.** The guardrail block is a terminal transition; blocked requests sit in `BLOCKED` until an operator unblocks via `POST /api/requests/{id}/unblock`. A `PASSENGER_REJECTED` is also terminal — the passenger's explicit refusal is the audit record.
- **Simulator throughput.** `RequestSimulator` drips one request every 30 s; the system can comfortably process each request end-to-end inside that window with mock or real LLMs.
