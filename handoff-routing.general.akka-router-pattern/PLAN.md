# PLAN — akka-router-pattern

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

  Sim[TaskSimulator]:::ta
  Queue[TaskQueue]:::ese
  QC[QueueConsumer]:::cons
  Cls[ClassifierAgent]:::agent
  Guard[RoutingGuardrail]:::agent
  Content[ContentSpecialist]:::autonomous
  Code[CodeSpecialist]:::autonomous
  Data[DataSpecialist]:::autonomous
  Judge[RoutingJudge]:::agent
  WF[RouterWorkflow]:::wf
  Entity[RequestEntity]:::ese
  View[RequestView]:::view
  Scorer[RoutingEvalScorer]:::cons
  API[RouterEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| QC
  QC -->|register + start workflow| Entity
  QC -->|start workflow| WF
  WF -->|classify| Cls
  WF -->|guardrail check| Guard
  WF -->|EXECUTE task| Content
  WF -->|EXECUTE task| Code
  WF -->|EXECUTE task| Data
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

## Interaction sequence — J1 (content happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'actorTextColor':'#ffffff','noteTextColor':'#ffffff','sequenceNumberColor':'#000'
}}}%%
sequenceDiagram
  autonumber
  participant Sim as TaskSimulator
  participant Q as TaskQueue
  participant QC as QueueConsumer
  participant E as RequestEntity
  participant W as RouterWorkflow
  participant Cls as ClassifierAgent
  participant Grd as RoutingGuardrail
  participant Cnt as ContentSpecialist
  participant Sc as RoutingEvalScorer
  participant J as RoutingJudge

  Sim->>Q: receive(TaskRequest)
  Q->>QC: InboundTaskReceived
  QC->>E: registerRequest
  QC->>W: start(requestId, request)
  W->>Cls: classify(request)
  Cls-->>W: ClassificationDecision{CONTENT}
  W->>E: recordClassification(decision) [emits ClassificationDecided]
  E->>Sc: ClassificationDecided event
  Sc->>J: score(request, decision)
  J-->>Sc: RoutingScore
  Sc->>E: recordRoutingScore [emits RoutingScored]
  W->>Grd: check(request, decision)
  Grd-->>W: GuardrailVerdict{ALLOWED}
  W->>E: recordGuardrailVerdict [emits GuardrailVerdictAttached]
  W->>E: recordRouting(CONTENT) [emits RequestRouted]
  W->>Cnt: runSingleTask(EXECUTE, prompt)
  Cnt-->>W: TaskResult
  W->>E: recordExecutionStart [emits ExecutionStarted]
  W->>E: publish(result) [emits ResultPublished, status COMPLETED]
```

The eval-event sequence (steps 7–10) runs concurrently with the workflow's continuation — `RoutingEvalScorer` is a Consumer reading the entity's event stream, independent of `RouterWorkflow`. Both writes target the same `RequestEntity`; the entity's commands are idempotent on `requestId`.

## State machine — `RequestEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> CLASSIFIED: ClassificationDecided
  CLASSIFIED --> GUARDRAIL_PASSED: GuardrailVerdictAttached (ALLOWED)
  CLASSIFIED --> BLOCKED: GuardrailVerdictAttached (DENIED)
  CLASSIFIED --> UNROUTABLE: domain = UNKNOWN
  GUARDRAIL_PASSED --> ROUTED_CONTENT: domain = CONTENT
  GUARDRAIL_PASSED --> ROUTED_CODE: domain = CODE
  GUARDRAIL_PASSED --> ROUTED_DATA: domain = DATA
  ROUTED_CONTENT --> EXECUTING: ExecutionStarted
  ROUTED_CODE --> EXECUTING: ExecutionStarted
  ROUTED_DATA --> EXECUTING: ExecutionStarted
  EXECUTING --> COMPLETED: ResultPublished
  BLOCKED --> COMPLETED: operator unblock
  COMPLETED --> [*]
  UNROUTABLE --> [*]
```

The `RoutingScored` event does not change `status`; it attaches the eval result. The state machine therefore treats it as a no-op transition (omitted from the diagram for clarity).

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  RequestEntity ||--o{ RequestRegistered : emits
  RequestEntity ||--o{ ClassificationDecided : emits
  RequestEntity ||--o{ GuardrailVerdictAttached : emits
  RequestEntity ||--o{ RequestRouted : emits
  RequestEntity ||--o{ ExecutionStarted : emits
  RequestEntity ||--o{ RequestBlocked : emits
  RequestEntity ||--o{ ResultPublished : emits
  RequestEntity ||--o{ RequestUnroutable : emits
  RequestEntity ||--o{ RoutingScored : emits
  RequestView }o--|| RequestEntity : projects
  TaskQueue ||--o{ InboundTaskReceived : emits
  QueueConsumer }o--|| TaskQueue : subscribes
  RoutingEvalScorer }o--|| RequestEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `TaskSimulator` | `application/TaskSimulator.java` |
| `TaskQueue` | `application/TaskQueue.java` |
| `QueueConsumer` | `application/QueueConsumer.java` |
| `ClassifierAgent` | `application/ClassifierAgent.java` |
| `RoutingGuardrail` | `application/RoutingGuardrail.java` |
| `ContentSpecialist` | `application/ContentSpecialist.java` |
| `CodeSpecialist` | `application/CodeSpecialist.java` |
| `DataSpecialist` | `application/DataSpecialist.java` |
| `RoutingJudge` | `application/RoutingJudge.java` |
| `RouterWorkflow` | `application/RouterWorkflow.java` |
| `RequestEntity` | `application/RequestEntity.java` (state in `domain/Request.java`, events in `domain/RequestEvent.java`) |
| `RequestView` | `application/RequestView.java` |
| `RoutingEvalScorer` | `application/RoutingEvalScorer.java` |
| `RouterEndpoint` | `api/RouterEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/RouterTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `classifyStep` 20 s, `guardrailStep` 20 s, `contentStep` / `codeStep` / `dataStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the request to `UNROUTABLE` with the failure reason captured.
- **Idempotency.** Every per-request primitive is keyed by `requestId`: `RequestEntity` id is `requestId`; `RouterWorkflow` id is `requestId`; agent sessions for `ClassifierAgent`, `RoutingGuardrail`, and `RoutingJudge` use `requestId`. Duplicate queue consumer events fold into a single workflow start (workflow start is idempotent per id).
- **Race between eval and workflow.** `RoutingEvalScorer` (Consumer) and `RouterWorkflow` both append events to the same `RequestEntity`. Order is not guaranteed but does not matter: `RoutingScored` only mutates `routingScore`, never `status`. The view materialises both events independently.
- **No saga compensation.** Once the specialist returns its `TaskResult`, the workflow publishes unconditionally. There is no rollback path — a blocked request sits in `BLOCKED` until an operator unblocks via `POST /api/requests/{id}/unblock`.
- **Guardrail precedes specialist.** The `RoutingGuardrail` runs in `guardrailStep` immediately after classification and before any specialist is selected. A blocked request never reaches a specialist; the specialist sees no evidence of it.
- **Simulator throughput.** `TaskSimulator` drips one request every 30 s; the system can comfortably process each request end-to-end inside that window with mock or real LLMs.
