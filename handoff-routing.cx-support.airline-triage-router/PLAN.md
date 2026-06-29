# PLAN — airline-triage-router

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

  Feeder[RequestFeeder]:::ta
  Queue[PassengerRequestQueue]:::ese
  Sanit[PnrSanitizer]:::cons
  Router[IntentRouter]:::agent
  Booking[BookingSpecialist]:::autonomous
  Change[ChangeSpecialist]:::autonomous
  Baggage[BaggageSpecialist]:::autonomous
  Status[StatusSpecialist]:::autonomous
  Judge[RoutingJudge]:::agent
  ToolGuard[ToolCallGuardrail]:::agent
  RespGuard[ResponseGuardrail]:::agent
  WF[FlightRequestWorkflow]:::wf
  Entity[FlightRequestEntity]:::ese
  View[FlightRequestView]:::view
  Scorer[RoutingEvalScorer]:::cons
  API[AirlineEndpoint]:::ep
  App[AppEndpoint]:::ep

  Feeder -.->|every 30s| Queue
  Queue -.->|subscribes| Sanit
  Sanit -->|register + sanitize| Entity
  Sanit -->|start workflow| WF
  WF -->|classify| Router
  WF -->|RESOLVE task| Booking
  WF -->|RESOLVE task| Change
  WF -->|RESOLVE task| Baggage
  WF -->|RESOLVE task| Status
  WF -->|check tool call| ToolGuard
  WF -->|check draft| RespGuard
  WF -->|emit events| Entity
  Entity -.->|projects| View
  Entity -.->|subscribes| Scorer
  Scorer -->|score| Judge
  Scorer -->|recordRoutingScore| Entity
  API -->|unblock / submit| Entity
  API -->|query / SSE| View
  App -->|static UI + metadata| API
```

Solid arrows = synchronous component calls. Dashed arrows = event subscriptions and scheduler ticks.

## Interaction sequence — J1 (booking happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'actorTextColor':'#ffffff','noteTextColor':'#ffffff','sequenceNumberColor':'#000'
}}}%%
sequenceDiagram
  autonumber
  participant F as RequestFeeder
  participant Q as PassengerRequestQueue
  participant P as PnrSanitizer
  participant E as FlightRequestEntity
  participant W as FlightRequestWorkflow
  participant R as IntentRouter
  participant B as BookingSpecialist
  participant TG as ToolCallGuardrail
  participant RG as ResponseGuardrail
  participant Sc as RoutingEvalScorer
  participant J as RoutingJudge

  F->>Q: receive(PassengerRequest)
  Q->>P: InboundPassengerRequestReceived
  P->>E: registerIncoming + attachSanitized
  P->>W: start(requestId, sanitized)
  W->>R: classify(sanitized)
  R-->>W: IntentDecision{BOOKING}
  W->>E: recordIntent(decision) [emits IntentClassified]
  E->>Sc: IntentClassified event
  Sc->>J: score(sanitized, decision)
  J-->>Sc: RoutingScore
  Sc->>E: recordRoutingScore [emits RoutingScored]
  W->>E: recordRouting(BOOKING) [emits RequestRouted]
  W->>B: runSingleTask(RESOLVE, prompt)
  B->>TG: check(sanitized, proposedAction)
  TG-->>B: ToolCallVerdict{allowed=true}
  B-->>W: AirlineResolution
  W->>E: recordDraft(resolution) [emits ResolutionDrafted]
  W->>RG: check(sanitized, resolution)
  RG-->>W: ResponseVerdict{allowed=true}
  W->>E: publish(resolution) [emits ResponsePublished, status RESOLVED]
```

The eval-event sequence (steps 7–10) runs concurrently with the workflow's continuation — `RoutingEvalScorer` is a Consumer reading the entity's event stream, independent of `FlightRequestWorkflow`. Both writes target the same `FlightRequestEntity`; the entity's commands are idempotent on `requestId`.

## State machine — `FlightRequestEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: RequestSanitized
  SANITIZED --> CLASSIFIED: IntentClassified
  CLASSIFIED --> ROUTED_BOOKING: intent = BOOKING
  CLASSIFIED --> ROUTED_CHANGE: intent = CHANGE
  CLASSIFIED --> ROUTED_BAGGAGE: intent = BAGGAGE
  CLASSIFIED --> ROUTED_STATUS: intent = STATUS
  CLASSIFIED --> UNRESOLVED: intent = UNCLEAR
  ROUTED_BOOKING --> RESOLUTION_DRAFTED: ResolutionDrafted
  ROUTED_CHANGE --> RESOLUTION_DRAFTED: ResolutionDrafted
  ROUTED_BAGGAGE --> RESOLUTION_DRAFTED: ResolutionDrafted
  ROUTED_STATUS --> RESOLUTION_DRAFTED: ResolutionDrafted
  RESOLUTION_DRAFTED --> RESOLVED: responseVerdict.allowed
  RESOLUTION_DRAFTED --> BLOCKED: responseVerdict.rejected
  ROUTED_CHANGE --> BLOCKED: toolCallVerdict.denied
  BLOCKED --> RESOLVED: operator unblock
  RESOLVED --> [*]
  UNRESOLVED --> [*]
```

The `RoutingScored` event does not change `status`; it attaches the eval result. The state machine omits this as a no-op transition.

A `BLOCKED` state can arise from either a denied tool call (before the specialist finishes) or a blocked response (after the specialist returns a draft). Both surface the same way to the operator.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  FlightRequestEntity ||--o{ RequestRegistered : emits
  FlightRequestEntity ||--o{ RequestSanitized : emits
  FlightRequestEntity ||--o{ IntentClassified : emits
  FlightRequestEntity ||--o{ RequestRouted : emits
  FlightRequestEntity ||--o{ ResolutionDrafted : emits
  FlightRequestEntity ||--o{ ToolCallVerdictAttached : emits
  FlightRequestEntity ||--o{ ResponseVerdictAttached : emits
  FlightRequestEntity ||--o{ ResponsePublished : emits
  FlightRequestEntity ||--o{ ResponseBlocked : emits
  FlightRequestEntity ||--o{ RequestUnresolved : emits
  FlightRequestEntity ||--o{ RoutingScored : emits
  FlightRequestView }o--|| FlightRequestEntity : projects
  PassengerRequestQueue ||--o{ InboundPassengerRequestReceived : emits
  PnrSanitizer }o--|| PassengerRequestQueue : subscribes
  RoutingEvalScorer }o--|| FlightRequestEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RequestFeeder` | `application/RequestFeeder.java` |
| `PassengerRequestQueue` | `application/PassengerRequestQueue.java` |
| `PnrSanitizer` | `application/PnrSanitizer.java` |
| `IntentRouter` | `application/IntentRouter.java` |
| `BookingSpecialist` | `application/BookingSpecialist.java` |
| `ChangeSpecialist` | `application/ChangeSpecialist.java` |
| `BaggageSpecialist` | `application/BaggageSpecialist.java` |
| `StatusSpecialist` | `application/StatusSpecialist.java` |
| `RoutingJudge` | `application/RoutingJudge.java` |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `ResponseGuardrail` | `application/ResponseGuardrail.java` |
| `FlightRequestWorkflow` | `application/FlightRequestWorkflow.java` |
| `FlightRequestEntity` | `application/FlightRequestEntity.java` (state in `domain/FlightRequest.java`, events in `domain/FlightRequestEvent.java`) |
| `FlightRequestView` | `application/FlightRequestView.java` |
| `RoutingEvalScorer` | `application/RoutingEvalScorer.java` |
| `AirlineEndpoint` | `api/AirlineEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/AirlineTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `classifyStep` 20 s, `responseGuardrailStep` 20 s, all specialist steps and `publishStep` 60 s. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the request to `UNRESOLVED` with the failure reason captured.
- **Idempotency.** Every per-request primitive is keyed by `requestId`: `FlightRequestEntity` id is `requestId`; `FlightRequestWorkflow` id is `requestId`; agent sessions for `IntentRouter`, `RoutingJudge`, `ToolCallGuardrail`, and `ResponseGuardrail` use `requestId`. Duplicate sanitize events fold into a single workflow start.
- **Two guardrail layers.** A before-tool-call denial and a response guardrail rejection both write to `BLOCKED`. They are not mutually exclusive but in practice only one fires per request: if the tool-call is denied, the specialist returns `outcome=ESCALATED` and the response guardrail sees a safe draft; if the tool-call is allowed, the response guardrail is the next gate.
- **Race between eval and workflow.** `RoutingEvalScorer` and `FlightRequestWorkflow` both append events to the same entity. The `RoutingScored` event only mutates `routingScore`, never `status`. The view materialises both independently.
- **Four-way branch.** The four intent categories produce four `ROUTED_*` states. They all converge at `RESOLUTION_DRAFTED` once the specialist returns, then share the single response guardrail step. The workflow code has one guard branch per intent; the post-draft path is shared.
- **Simulator throughput.** `RequestFeeder` drips one request every 30 s. With status-specialist and intent-router calls being fast, and booking/change/baggage at 60 s timeout, the system can process each request end-to-end inside that window on mock or real LLMs.
