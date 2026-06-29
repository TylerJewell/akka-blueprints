# PLAN — core-semantic-router

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

  Sim[QuerySimulator]:::ta
  Queue[QueryQueue]:::ese
  Sanit[PiiSanitizer]:::cons
  Router[IntentRouterAgent]:::agent
  Guard[RoutingGuardrail]:::agent
  HR[HrSpecialist]:::autonomous
  Fin[FinanceSpecialist]:::autonomous
  Judge[RoutingJudge]:::agent
  WF[RouterWorkflow]:::wf
  Entity[QueryEntity]:::ese
  View[QueryView]:::view
  Scorer[RoutingEvalScorer]:::cons
  API[RouterEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| Sanit
  Sanit -->|register + sanitize| Entity
  Sanit -->|start workflow| WF
  WF -->|classify| Router
  WF -->|check routing| Guard
  WF -->|ANSWER task| HR
  WF -->|ANSWER task| Fin
  WF -->|emit events| Entity
  Entity -.->|projects| View
  Entity -.->|subscribes| Scorer
  Scorer -->|score| Judge
  Scorer -->|recordRoutingScore| Entity
  API -->|submit| Queue
  API -->|query / SSE| View
  App -->|static UI + metadata| API
```

Solid arrows = synchronous component calls. Dashed arrows = event subscriptions and scheduler ticks.

## Interaction sequence — J1 (HR happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'actorTextColor':'#ffffff','noteTextColor':'#ffffff','sequenceNumberColor':'#000'
}}}%%
sequenceDiagram
  autonumber
  participant Sim as QuerySimulator
  participant Q as QueryQueue
  participant S as PiiSanitizer
  participant E as QueryEntity
  participant W as RouterWorkflow
  participant R as IntentRouterAgent
  participant G as RoutingGuardrail
  participant HR as HrSpecialist
  participant Sc as RoutingEvalScorer
  participant J as RoutingJudge

  Sim->>Q: receive(IncomingQuery)
  Q->>S: InboundQueryReceived
  S->>E: registerIncoming + attachSanitized
  S->>W: start(queryId, sanitized)
  W->>R: classify(sanitized)
  R-->>W: RoutingDecision{HR}
  W->>E: recordClassification(decision) [emits IntentClassified]
  W->>G: check(decision, callerCtx)
  G-->>W: GuardrailVerdict{authorized=true}
  W->>E: recordRouting(HR) [emits IntentRouted]
  E->>Sc: IntentRouted event
  Sc->>J: score(sanitized, decision)
  J-->>Sc: RoutingScore
  Sc->>E: recordRoutingScore [emits RoutingScored]
  W->>HR: runSingleTask(ANSWER, prompt)
  HR-->>W: QueryAnswer
  W->>E: recordAnswer(answer) [emits QueryAnswered]
  W->>E: publish(answer) [emits AnswerPublished, status ANSWERED]
```

The eval-event sequence (steps 10–13) runs concurrently with the workflow's continuation — `RoutingEvalScorer` is a Consumer reading the entity's event stream, independent of `RouterWorkflow`. Both writes target the same `QueryEntity`; the entity's commands are idempotent on `queryId`.

## State machine — `QueryEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: QuerySanitized
  SANITIZED --> CLASSIFIED: IntentClassified
  CLASSIFIED --> AUTHORIZED: guardrail.authorized
  CLASSIFIED --> BLOCKED: guardrail.rejected
  AUTHORIZED --> ROUTED_HR: domain = HR
  AUTHORIZED --> ROUTED_FINANCE: domain = FINANCE
  AUTHORIZED --> ESCALATED: domain = AMBIGUOUS
  ROUTED_HR --> ANSWERED: QueryAnswered
  ROUTED_FINANCE --> ANSWERED: QueryAnswered
  ANSWERED --> [*]
  BLOCKED --> [*]
  ESCALATED --> [*]
```

The `RoutingScored` event does not change `status`; it attaches the eval result. The state machine treats it as a no-op transition (omitted from the diagram for clarity).

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  QueryEntity ||--o{ QueryRegistered : emits
  QueryEntity ||--o{ QuerySanitized : emits
  QueryEntity ||--o{ IntentClassified : emits
  QueryEntity ||--o{ GuardrailVerdictAttached : emits
  QueryEntity ||--o{ IntentRouted : emits
  QueryEntity ||--o{ QueryAnswered : emits
  QueryEntity ||--o{ QueryBlocked : emits
  QueryEntity ||--o{ QueryEscalated : emits
  QueryEntity ||--o{ RoutingScored : emits
  QueryView }o--|| QueryEntity : projects
  QueryQueue ||--o{ InboundQueryReceived : emits
  PiiSanitizer }o--|| QueryQueue : subscribes
  RoutingEvalScorer }o--|| QueryEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QuerySimulator` | `application/QuerySimulator.java` |
| `QueryQueue` | `application/QueryQueue.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `IntentRouterAgent` | `application/IntentRouterAgent.java` |
| `RoutingGuardrail` | `application/RoutingGuardrail.java` |
| `HrSpecialist` | `application/HrSpecialist.java` |
| `FinanceSpecialist` | `application/FinanceSpecialist.java` |
| `RoutingJudge` | `application/RoutingJudge.java` |
| `RouterWorkflow` | `application/RouterWorkflow.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `QueryView` | `application/QueryView.java` |
| `RoutingEvalScorer` | `application/RoutingEvalScorer.java` |
| `RouterEndpoint` | `api/RouterEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/RouterTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `classifyStep` 20 s, `guardrailStep` 20 s, `hrStep` / `financeStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the query to `ESCALATED` with the failure reason captured.
- **Idempotency.** Every per-query primitive is keyed by `queryId`: `QueryEntity` id is `queryId`; `RouterWorkflow` id is `queryId`; agent sessions for `IntentRouterAgent`, `RoutingGuardrail`, and `RoutingJudge` use `queryId`. Duplicate sanitize events fold into a single workflow start (workflow start is idempotent per id).
- **Race between eval and workflow.** `RoutingEvalScorer` (Consumer) and `RouterWorkflow` both append events to the same `QueryEntity`. Order is not guaranteed but does not matter: `RoutingScored` only mutates `routingScore`, never `status`. The view materialises both events independently.
- **Guardrail position.** The `RoutingGuardrail` check happens before the specialist is invoked. This is the distinction from a before-agent-response guardrail — no specialist runs at all when the routing is blocked. The query lands in `BLOCKED` and no `QueryAnswer` is ever produced.
- **No saga compensation.** The handoff is a single-direction transfer; once the specialist returns its `QueryAnswer`, the workflow publishes. There is no rollback path.
- **No HITL on the happy path.** The system only surfaces blocked queries for operator attention; everything else flows to `ANSWERED` autonomously.
- **Simulator throughput.** `QuerySimulator` drips one query every 30 s; the system can comfortably process each query end-to-end inside that window with mock or real LLMs.
