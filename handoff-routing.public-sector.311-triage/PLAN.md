# PLAN — 311-triage

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
  Triage[TriageAgent]:::agent
  Guard[RouteGuardrail]:::agent
  PW[PublicWorksSpecialist]:::autonomous
  Permits[PermitsSpecialist]:::autonomous
  Judge[TriageJudge]:::agent
  WF[RequestWorkflow]:::wf
  Entity[ServiceRequestEntity]:::ese
  View[RequestView]:::view
  Scorer[TriageEvalScorer]:::cons
  API[TriageEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| Sanit
  Sanit -->|register + sanitize| Entity
  Sanit -->|start workflow| WF
  WF -->|triage| Triage
  WF -->|audit route| Guard
  WF -->|HANDLE task| PW
  WF -->|HANDLE task| Permits
  WF -->|emit events| Entity
  Entity -.->|projects| View
  Entity -.->|subscribes| Scorer
  Scorer -->|score| Judge
  Scorer -->|recordTriageScore| Entity
  API -->|review / submit| Entity
  API -->|query / SSE| View
  App -->|static UI + metadata| API
```

Solid arrows = synchronous component calls. Dashed arrows = event subscriptions and scheduler ticks.

## Interaction sequence — J1 (public-works happy path)

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
  participant E as ServiceRequestEntity
  participant W as RequestWorkflow
  participant T as TriageAgent
  participant G as RouteGuardrail
  participant PW as PublicWorksSpecialist
  participant Sc as TriageEvalScorer
  participant J as TriageJudge

  Sim->>Q: receive(InboundServiceRequest)
  Q->>S: InboundRequestReceived
  S->>E: registerInbound + attachSanitized
  S->>W: start(requestId, sanitized)
  W->>T: triage(sanitized)
  T-->>W: TriageDecision{PUBLIC_WORKS}
  W->>E: recordTriage(decision) [emits TriageDecided]
  E->>Sc: TriageDecided event
  Sc->>J: score(sanitized, decision)
  J-->>Sc: TriageScore
  Sc->>E: recordTriageScore [emits TriageScored]
  W->>G: check(sanitized, decision)
  G-->>W: RouteVerdict{approved=true}
  W->>E: recordRouteVerdict + recordRouting(PUBLIC_WORKS) [emits RouteVerdictAttached, ServiceRequestRouted]
  W->>PW: runSingleTask(HANDLE, prompt)
  PW-->>W: DepartmentResponse
  W->>E: recordDraft(response) [emits ResponseDrafted]
  W->>E: publish(response) [emits ResponsePublished, status RESOLVED]
```

The eval-event sequence (steps 8–11) runs concurrently with the workflow's continuation — `TriageEvalScorer` is a Consumer reading the entity's event stream, independent of `RequestWorkflow`. Both writes target the same `ServiceRequestEntity`; the entity's commands are idempotent on `requestId`.

## State machine — `ServiceRequestEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: ServiceRequestSanitized
  SANITIZED --> TRIAGED: TriageDecided
  TRIAGED --> ROUTE_APPROVED: RouteVerdict approved
  TRIAGED --> FLAGGED_FOR_REVIEW: RouteVerdict flagged
  TRIAGED --> ESCALATED: category = UNCLEAR
  ROUTE_APPROVED --> ROUTED_PUBLIC_WORKS: category = PUBLIC_WORKS
  ROUTE_APPROVED --> ROUTED_PERMITS_ZONING: category = PERMITS_ZONING
  ROUTED_PUBLIC_WORKS --> RESPONSE_DRAFTED: ResponseDrafted
  ROUTED_PERMITS_ZONING --> RESPONSE_DRAFTED: ResponseDrafted
  RESPONSE_DRAFTED --> RESOLVED: ResponsePublished
  FLAGGED_FOR_REVIEW --> ROUTE_APPROVED: reviewer approves
  FLAGGED_FOR_REVIEW --> ESCALATED: reviewer escalates
  RESOLVED --> [*]
  ESCALATED --> [*]
```

The `TriageScored` event does not change `status`; it attaches the eval result. The state machine treats it as a no-op transition (omitted for clarity).

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  ServiceRequestEntity ||--o{ ServiceRequestRegistered : emits
  ServiceRequestEntity ||--o{ ServiceRequestSanitized : emits
  ServiceRequestEntity ||--o{ TriageDecided : emits
  ServiceRequestEntity ||--o{ RouteVerdictAttached : emits
  ServiceRequestEntity ||--o{ ServiceRequestRouted : emits
  ServiceRequestEntity ||--o{ ResponseDrafted : emits
  ServiceRequestEntity ||--o{ ResponsePublished : emits
  ServiceRequestEntity ||--o{ RequestFlagged : emits
  ServiceRequestEntity ||--o{ RequestEscalated : emits
  ServiceRequestEntity ||--o{ TriageScored : emits
  RequestView }o--|| ServiceRequestEntity : projects
  RequestQueue ||--o{ InboundRequestReceived : emits
  PiiSanitizer }o--|| RequestQueue : subscribes
  TriageEvalScorer }o--|| ServiceRequestEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RequestSimulator` | `application/RequestSimulator.java` |
| `RequestQueue` | `application/RequestQueue.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `TriageAgent` | `application/TriageAgent.java` |
| `RouteGuardrail` | `application/RouteGuardrail.java` |
| `PublicWorksSpecialist` | `application/PublicWorksSpecialist.java` |
| `PermitsSpecialist` | `application/PermitsSpecialist.java` |
| `TriageJudge` | `application/TriageJudge.java` |
| `RequestWorkflow` | `application/RequestWorkflow.java` |
| `ServiceRequestEntity` | `application/ServiceRequestEntity.java` (state in `domain/ServiceRequest.java`, events in `domain/ServiceRequestEvent.java`) |
| `RequestView` | `application/RequestView.java` |
| `TriageEvalScorer` | `application/TriageEvalScorer.java` |
| `TriageEndpoint` | `api/TriageEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/ServiceTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `triageStep` 20 s, `routeGuardrailStep` 20 s, `publicWorksStep` / `permitsStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the request to `ESCALATED` with the failure reason captured.
- **Idempotency.** Every per-request primitive is keyed by `requestId`: `ServiceRequestEntity` id is `requestId`; `RequestWorkflow` id is `requestId`; agent sessions for `TriageAgent`, `RouteGuardrail`, `TriageJudge` use `requestId`. Duplicate sanitize events fold into a single workflow start (workflow start is idempotent per id).
- **Race between eval and workflow.** `TriageEvalScorer` (Consumer) and `RequestWorkflow` both append events to the same `ServiceRequestEntity`. Order is not guaranteed but does not matter: `TriageScored` only mutates `triageScore`, never `status`. The view materialises both events independently.
- **No saga compensation.** The handoff is a single-direction transfer of ownership; once the specialist returns its `DepartmentResponse`, the workflow publishes to `RESOLVED`. There is no rollback path — a flagged route sits in `FLAGGED_FOR_REVIEW` until a reviewer acts via `POST /api/requests/{id}/review`.
- **No HITL on the happy path.** The system only waits for a human when the route guardrail flags; everything else flows through to `RESOLVED` autonomously.
- **Simulator throughput.** `RequestSimulator` drips one request every 30 s; the system can process each request end-to-end inside that window with mock or real LLMs.
