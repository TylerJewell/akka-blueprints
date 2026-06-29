# PLAN — auto-insurance-agent

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
  Triage[ClaimTriageAgent]:::agent
  Claims[ClaimsSpecialist]:::autonomous
  Policy[PolicySpecialist]:::autonomous
  Rewards[RewardsSpecialist]:::autonomous
  Roadside[RoadsideSpecialist]:::autonomous
  Judge[TriageJudge]:::agent
  Guard[ResponseGuardrail]:::agent
  WF[InsuranceWorkflow]:::wf
  Entity[RequestEntity]:::ese
  View[RequestView]:::view
  Scorer[TriageEvalScorer]:::cons
  API[InsuranceEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| Sanit
  Sanit -->|register + sanitize| Entity
  Sanit -->|start workflow| WF
  WF -->|triage| Triage
  WF -->|HANDLE task| Claims
  WF -->|HANDLE task| Policy
  WF -->|HANDLE task| Rewards
  WF -->|HANDLE task| Roadside
  WF -->|check draft| Guard
  WF -->|emit events| Entity
  Entity -.->|projects| View
  Entity -.->|subscribes| Scorer
  Scorer -->|score| Judge
  Scorer -->|recordTriageScore| Entity
  API -->|unblock / submit| Entity
  API -->|query / SSE| View
  App -->|static UI + metadata| API
```

Solid arrows = synchronous component calls. Dashed arrows = event subscriptions and scheduler ticks.

## Interaction sequence — J1 (claim happy path)

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
  participant W as InsuranceWorkflow
  participant T as ClaimTriageAgent
  participant C as ClaimsSpecialist
  participant G as ResponseGuardrail
  participant Sc as TriageEvalScorer
  participant J as TriageJudge

  Sim->>Q: receive(IncomingRequest)
  Q->>S: InboundRequestReceived
  S->>E: registerIncoming + attachSanitized
  S->>W: start(requestId, sanitized)
  W->>T: triage(sanitized)
  T-->>W: TriageDecision{CLAIM}
  W->>E: recordTriage(decision) [emits TriageDecided]
  E->>Sc: TriageDecided event
  Sc->>J: score(sanitized, decision)
  J-->>Sc: TriageScore
  Sc->>E: recordTriageScore [emits TriageScored]
  W->>E: recordRouting(CLAIM) [emits RequestRouted]
  W->>C: runSingleTask(HANDLE, prompt)
  C-->>W: MemberResponse
  W->>E: recordDraft(response) [emits ResponseDrafted]
  W->>G: check(sanitized, response)
  G-->>W: GuardrailVerdict{allowed=true}
  W->>E: publish(response) [emits ResponsePublished, status RESOLVED]
```

The eval-event sequence (steps 7–10) runs concurrently with the workflow's continuation — `TriageEvalScorer` is a Consumer reading the entity's event stream, independent of `InsuranceWorkflow`. Both writes target the same `RequestEntity`; the entity's commands are idempotent on `requestId`.

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
  SANITIZED --> TRIAGED: TriageDecided
  TRIAGED --> ROUTED_CLAIM: category = CLAIM
  TRIAGED --> ROUTED_POLICY: category = POLICY
  TRIAGED --> ROUTED_REWARDS: category = REWARDS
  TRIAGED --> ROUTED_ROADSIDE: category = ROADSIDE
  TRIAGED --> ESCALATED: category = UNCLEAR
  ROUTED_CLAIM --> RESPONSE_DRAFTED: ResponseDrafted
  ROUTED_POLICY --> RESPONSE_DRAFTED: ResponseDrafted
  ROUTED_REWARDS --> RESPONSE_DRAFTED: ResponseDrafted
  ROUTED_ROADSIDE --> RESPONSE_DRAFTED: ResponseDrafted
  RESPONSE_DRAFTED --> RESOLVED: guardrail.allowed
  RESPONSE_DRAFTED --> BLOCKED: guardrail.rejected
  BLOCKED --> RESOLVED: supervisor unblock
  RESOLVED --> [*]
  ESCALATED --> [*]
```

The `TriageScored` event does not change `status`; it attaches the eval result. The state machine treats it as a no-op transition (omitted from the diagram for clarity).

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  RequestEntity ||--o{ RequestRegistered : emits
  RequestEntity ||--o{ RequestSanitized : emits
  RequestEntity ||--o{ TriageDecided : emits
  RequestEntity ||--o{ RequestRouted : emits
  RequestEntity ||--o{ ResponseDrafted : emits
  RequestEntity ||--o{ GuardrailVerdictAttached : emits
  RequestEntity ||--o{ ResponsePublished : emits
  RequestEntity ||--o{ ResponseBlocked : emits
  RequestEntity ||--o{ RequestEscalated : emits
  RequestEntity ||--o{ TriageScored : emits
  RequestView }o--|| RequestEntity : projects
  RequestQueue ||--o{ InboundRequestReceived : emits
  PiiSanitizer }o--|| RequestQueue : subscribes
  TriageEvalScorer }o--|| RequestEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RequestSimulator` | `application/RequestSimulator.java` |
| `RequestQueue` | `application/RequestQueue.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `ClaimTriageAgent` | `application/ClaimTriageAgent.java` |
| `ClaimsSpecialist` | `application/ClaimsSpecialist.java` |
| `PolicySpecialist` | `application/PolicySpecialist.java` |
| `RewardsSpecialist` | `application/RewardsSpecialist.java` |
| `RoadsideSpecialist` | `application/RoadsideSpecialist.java` |
| `TriageJudge` | `application/TriageJudge.java` |
| `ResponseGuardrail` | `application/ResponseGuardrail.java` |
| `InsuranceWorkflow` | `application/InsuranceWorkflow.java` |
| `RequestEntity` | `application/RequestEntity.java` (state in `domain/MemberRequest.java`, events in `domain/RequestEvent.java`) |
| `RequestView` | `application/RequestView.java` |
| `TriageEvalScorer` | `application/TriageEvalScorer.java` |
| `InsuranceEndpoint` | `api/InsuranceEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/InsuranceTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `triageStep` 20 s, `guardrailStep` 20 s, `claimsStep` / `policyStep` / `rewardsStep` / `roadsideStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the request to `ESCALATED` with the failure reason captured.
- **Idempotency.** Every per-request primitive is keyed by `requestId`: `RequestEntity` id is `requestId`; `InsuranceWorkflow` id is `requestId`; agent sessions for `ClaimTriageAgent`, `TriageJudge`, and `ResponseGuardrail` use `requestId`. Duplicate sanitize events fold into a single workflow start (workflow start is idempotent per id).
- **Race between eval and workflow.** `TriageEvalScorer` (Consumer) and `InsuranceWorkflow` both append events to the same `RequestEntity`. Order is not guaranteed but does not matter: `TriageScored` only mutates `triageScore`, never `status`. The view materialises both events independently.
- **No saga compensation.** The handoff is a single-direction transfer of ownership; once the specialist returns its `MemberResponse`, the workflow either publishes or blocks based on the guardrail verdict. A blocked draft sits in `BLOCKED` until a supervisor unblocks via `POST /api/requests/{id}/unblock`.
- **No HITL on the happy path.** The system only waits for a human when the guardrail blocks; all other paths flow through to `RESOLVED` autonomously.
- **Four-way fan-out.** The workflow routes to exactly one of four specialists per request; the others are never invoked. From the component graph's perspective the four branches all terminate at the same `guardrailStep`.
- **Simulator throughput.** `RequestSimulator` drips one request every 30 s; the system can comfortably process each request end-to-end inside that window with mock or real LLMs.
