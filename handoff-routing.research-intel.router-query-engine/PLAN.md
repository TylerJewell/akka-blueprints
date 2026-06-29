# PLAN — router-query-engine

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

  Sim[QuestionSimulator]:::ta
  Queue[QuestionQueue]:::ese
  QCons[QuestionConsumer]:::cons
  Router[RouterAgent]:::agent
  Structured[StructuredDataEngine]:::autonomous
  Semantic[SemanticSearchEngine]:::autonomous
  Judge[RoutingJudge]:::agent
  WF[QueryWorkflow]:::wf
  Entity[QueryEntity]:::ese
  View[QueryView]:::view
  Scorer[RoutingEvalScorer]:::cons
  API[QueryEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| QCons
  QCons -->|register + start workflow| Entity
  QCons -->|start workflow| WF
  WF -->|route| Router
  WF -->|ANSWER task| Structured
  WF -->|ANSWER task| Semantic
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

## Interaction sequence — J1 (structured-data happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'actorTextColor':'#ffffff','noteTextColor':'#ffffff','sequenceNumberColor':'#000'
}}}%%
sequenceDiagram
  autonumber
  participant Sim as QuestionSimulator
  participant Q as QuestionQueue
  participant C as QuestionConsumer
  participant E as QueryEntity
  participant W as QueryWorkflow
  participant R as RouterAgent
  participant SD as StructuredDataEngine
  participant Sc as RoutingEvalScorer
  participant J as RoutingJudge

  Sim->>Q: receive(ResearchQuestion)
  Q->>C: InboundQuestionReceived
  C->>E: registerQuestion
  C->>W: start(queryId, question)
  W->>R: route(question)
  R-->>W: RoutingDecision{STRUCTURED}
  W->>E: recordRouting(decision) [emits RoutingDecided]
  E->>Sc: RoutingDecided event
  Sc->>J: score(question, decision)
  J-->>Sc: RoutingScore
  Sc->>E: recordRoutingScore [emits RoutingScored]
  W->>E: recordRouted(STRUCTURED) [emits QueryRouted]
  W->>SD: runSingleTask(ANSWER, prompt)
  SD-->>W: Answer
  W->>E: recordAnswerDraft(answer) [emits AnswerDrafted]
  W->>E: publishAnswer(answer) [emits AnswerPublished, status ANSWERED]
```

The eval-event sequence (steps 7–10) runs concurrently with the workflow's continuation — `RoutingEvalScorer` is a Consumer reading the entity's event stream, independent of `QueryWorkflow`. Both writes target the same `QueryEntity`; the entity's commands are idempotent on `queryId`.

## State machine — `QueryEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> ROUTING: QueryRegistered
  ROUTING --> ROUTED_STRUCTURED: engineType = STRUCTURED
  ROUTING --> ROUTED_SEMANTIC: engineType = SEMANTIC
  ROUTING --> ESCALATED: engineType = UNCLEAR
  ROUTED_STRUCTURED --> ANSWERING: AnswerDrafted
  ROUTED_SEMANTIC --> ANSWERING: AnswerDrafted
  ANSWERING --> ANSWERED: AnswerPublished
  ANSWERED --> [*]
  ESCALATED --> [*]
```

The `RoutingScored` event does not change `status`; it attaches the eval result. The state machine therefore treats it as a no-op transition (omitted from the diagram for clarity).

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  QueryEntity ||--o{ QueryRegistered : emits
  QueryEntity ||--o{ RoutingDecided : emits
  QueryEntity ||--o{ QueryRouted : emits
  QueryEntity ||--o{ AnswerDrafted : emits
  QueryEntity ||--o{ AnswerPublished : emits
  QueryEntity ||--o{ QueryEscalated : emits
  QueryEntity ||--o{ RoutingScored : emits
  QueryView }o--|| QueryEntity : projects
  QuestionQueue ||--o{ InboundQuestionReceived : emits
  QuestionConsumer }o--|| QuestionQueue : subscribes
  RoutingEvalScorer }o--|| QueryEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QuestionSimulator` | `application/QuestionSimulator.java` |
| `QuestionQueue` | `application/QuestionQueue.java` |
| `QuestionConsumer` | `application/QuestionConsumer.java` |
| `RouterAgent` | `application/RouterAgent.java` |
| `StructuredDataEngine` | `application/StructuredDataEngine.java` |
| `SemanticSearchEngine` | `application/SemanticSearchEngine.java` |
| `RoutingJudge` | `application/RoutingJudge.java` |
| `QueryWorkflow` | `application/QueryWorkflow.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/Query.java`, events in `domain/QueryEvent.java`) |
| `QueryView` | `application/QueryView.java` |
| `RoutingEvalScorer` | `application/RoutingEvalScorer.java` |
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/QueryTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `routeStep` 20 s; `structuredStep` / `semanticStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the query to `ESCALATED` with the failure reason captured.
- **Idempotency.** Every per-query primitive is keyed by `queryId`: `QueryEntity` id is `queryId`; `QueryWorkflow` id is `queryId`; agent session for `RouterAgent` and `RoutingJudge` uses `queryId`. Duplicate consumer events fold into a single workflow start (workflow start is idempotent per id).
- **Race between eval and workflow.** `RoutingEvalScorer` (Consumer) and `QueryWorkflow` both append events to the same `QueryEntity`. Order is not guaranteed but does not matter: `RoutingScored` only mutates `routingScore`, never `status`. The view materialises both events independently.
- **No saga compensation.** The handoff is a single-direction transfer; once the chosen engine returns its `Answer`, the workflow publishes. There is no rollback path — escalated queries sit in `ESCALATED` as a permanent terminal state.
- **No HITL on the happy path.** The system publishes answers without operator review. This is the key distinction from a human-in-loop pattern; it is appropriate for research-intelligence use cases where the answer is advisory, not binding.
- **Simulator throughput.** `QuestionSimulator` drips one question every 30 s; the system can comfortably process each query end-to-end inside that window with mock or real LLMs.
