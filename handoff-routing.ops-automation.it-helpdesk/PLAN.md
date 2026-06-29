# PLAN — it-helpdesk

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
  Sanit[SecretSanitizer]:::cons
  Classify[ClassifierAgent]:::agent
  Access[AccessSpecialist]:::autonomous
  Infra[InfraSpecialist]:::autonomous
  Software[SoftwareSpecialist]:::autonomous
  Judge[RoutingJudge]:::agent
  Guard[TicketGuardrail]:::agent
  WF[HelpdeskWorkflow]:::wf
  Entity[RequestEntity]:::ese
  View[RequestView]:::view
  Scorer[RoutingEvalScorer]:::cons
  API[HelpdeskEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| Sanit
  Sanit -->|register + sanitize| Entity
  Sanit -->|start workflow| WF
  WF -->|classify| Classify
  WF -->|RESOLVE task| Access
  WF -->|RESOLVE task| Infra
  WF -->|RESOLVE task| Software
  WF -->|check proposed ticket| Guard
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

## Interaction sequence — J1 (access happy path)

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
  participant S as SecretSanitizer
  participant E as RequestEntity
  participant W as HelpdeskWorkflow
  participant C as ClassifierAgent
  participant A as AccessSpecialist
  participant G as TicketGuardrail
  participant Sc as RoutingEvalScorer
  participant J as RoutingJudge

  Sim->>Q: receive(IncomingRequest)
  Q->>S: InboundRequestReceived
  S->>E: registerIncoming + attachSanitized
  S->>W: start(requestId, sanitized)
  W->>C: classify(sanitized)
  C-->>W: ClassificationDecision{ACCESS}
  W->>E: recordClassification(decision) [emits RequestClassified]
  E->>Sc: RequestClassified event
  Sc->>J: score(sanitized, decision)
  J-->>Sc: RoutingScore
  Sc->>E: recordRoutingScore [emits RoutingScored]
  W->>E: recordRouting(ACCESS) [emits RequestRouted]
  W->>A: runSingleTask(RESOLVE, prompt)
  A-->>W: Resolution{proposedTicket: ProposedTicket}
  W->>E: recordDraft(resolution) [emits ResolutionDrafted]
  W->>G: check(sanitized, proposedTicket)
  G-->>W: GuardrailVerdict{allowed=true}
  W->>E: fileTicket(proposedTicket) [emits TicketFiled]
  W->>E: publish(resolution) [emits ResolutionPublished, status RESOLVED]
```

The eval-event sequence (steps 8–11) runs concurrently with the workflow's continuation — `RoutingEvalScorer` is a Consumer reading the entity's event stream. Both writes target the same `RequestEntity`; the entity's commands are idempotent on `requestId`.

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
  SANITIZED --> CLASSIFIED: RequestClassified
  CLASSIFIED --> ROUTED_ACCESS: category = ACCESS
  CLASSIFIED --> ROUTED_INFRASTRUCTURE: category = INFRASTRUCTURE
  CLASSIFIED --> ROUTED_SOFTWARE: category = SOFTWARE
  CLASSIFIED --> ESCALATED: category = UNCLEAR
  ROUTED_ACCESS --> RESOLUTION_DRAFTED: ResolutionDrafted
  ROUTED_INFRASTRUCTURE --> RESOLUTION_DRAFTED: ResolutionDrafted
  ROUTED_SOFTWARE --> RESOLUTION_DRAFTED: ResolutionDrafted
  RESOLUTION_DRAFTED --> RESOLVED: guardrail.allowed (or no ticket proposed)
  RESOLUTION_DRAFTED --> TICKET_BLOCKED: guardrail.rejected
  TICKET_BLOCKED --> RESOLVED: technician unblock
  RESOLVED --> [*]
  ESCALATED --> [*]
```

`RoutingScored` events do not change `status`; they attach the eval result. The state machine omits this as a no-op transition.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  RequestEntity ||--o{ RequestRegistered : emits
  RequestEntity ||--o{ RequestSanitized : emits
  RequestEntity ||--o{ RequestClassified : emits
  RequestEntity ||--o{ RequestRouted : emits
  RequestEntity ||--o{ ResolutionDrafted : emits
  RequestEntity ||--o{ GuardrailVerdictAttached : emits
  RequestEntity ||--o{ TicketFiled : emits
  RequestEntity ||--o{ TicketBlocked : emits
  RequestEntity ||--o{ ResolutionPublished : emits
  RequestEntity ||--o{ RequestEscalated : emits
  RequestEntity ||--o{ RoutingScored : emits
  RequestView }o--|| RequestEntity : projects
  RequestQueue ||--o{ InboundRequestReceived : emits
  SecretSanitizer }o--|| RequestQueue : subscribes
  RoutingEvalScorer }o--|| RequestEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RequestSimulator` | `application/RequestSimulator.java` |
| `RequestQueue` | `application/RequestQueue.java` |
| `SecretSanitizer` | `application/SecretSanitizer.java` |
| `ClassifierAgent` | `application/ClassifierAgent.java` |
| `AccessSpecialist` | `application/AccessSpecialist.java` |
| `InfraSpecialist` | `application/InfraSpecialist.java` |
| `SoftwareSpecialist` | `application/SoftwareSpecialist.java` |
| `RoutingJudge` | `application/RoutingJudge.java` |
| `TicketGuardrail` | `application/TicketGuardrail.java` |
| `HelpdeskWorkflow` | `application/HelpdeskWorkflow.java` |
| `RequestEntity` | `application/RequestEntity.java` (state in `domain/Request.java`, events in `domain/RequestEvent.java`) |
| `RequestView` | `application/RequestView.java` |
| `RoutingEvalScorer` | `application/RoutingEvalScorer.java` |
| `HelpdeskEndpoint` | `api/HelpdeskEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/HelpdeskTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `classifyStep` 20 s, `guardrailStep` 20 s, `accessStep` / `infraStep` / `softwareStep` / `fileStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)`, which transitions the request to `ESCALATED`.
- **Conditional guardrail.** The guardrail step is skipped when the specialist's `Resolution.proposedTicket` is empty. Resolutions that only provide a `responseBody` without a ticket write pass through without a guardrail call.
- **Idempotency.** Every per-request primitive is keyed by `requestId`: `RequestEntity` id is `requestId`; `HelpdeskWorkflow` id is `requestId`; agent sessions use `requestId`. Duplicate sanitize events fold into a single workflow start.
- **Race between eval and workflow.** `RoutingEvalScorer` (Consumer) and `HelpdeskWorkflow` both append events to the same `RequestEntity`. Order is not guaranteed but does not matter: `RoutingScored` only mutates `routingScore`, never `status`.
- **No HITL on the happy path.** The system only waits for a human when the guardrail blocks a proposed ticket; everything else flows to `RESOLVED` autonomously.
- **Three specialists, one task type.** All three specialists declare `capability(TaskAcceptance.of(RESOLVE))`; the workflow routes to the correct one by calling `forAutonomousAgent(<Specialist>.class, requestId)`. The task type is shared; the specialist class determines who handles it.
