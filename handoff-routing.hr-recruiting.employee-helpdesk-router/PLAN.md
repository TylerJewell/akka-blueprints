# PLAN — employee-helpdesk-router

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
  Sanit[PiiSanitizer]:::cons
  Router[TopicRouter]:::agent
  HR[HRSpecialist]:::autonomous
  IT[ITSpecialist]:::autonomous
  Policy[PolicySpecialist]:::autonomous
  Judge[RouteJudge]:::agent
  Guard[AnswerGuardrail]:::agent
  WF[HelpWorkflow]:::wf
  Entity[QuestionEntity]:::ese
  View[QuestionView]:::view
  Scorer[RouteEvalScorer]:::cons
  API[HelpdeskEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| Sanit
  Sanit -->|register + sanitize| Entity
  Sanit -->|start workflow| WF
  WF -->|route| Router
  WF -->|ANSWER task| HR
  WF -->|ANSWER task| IT
  WF -->|ANSWER task| Policy
  WF -->|check draft| Guard
  WF -->|emit events| Entity
  Entity -.->|projects| View
  Entity -.->|subscribes| Scorer
  Scorer -->|score| Judge
  Scorer -->|recordRouteScore| Entity
  API -->|unblock / submit| Entity
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
  participant Sim as QuestionSimulator
  participant Q as QuestionQueue
  participant S as PiiSanitizer
  participant E as QuestionEntity
  participant W as HelpWorkflow
  participant R as TopicRouter
  participant H as HRSpecialist
  participant G as AnswerGuardrail
  participant Sc as RouteEvalScorer
  participant J as RouteJudge

  Sim->>Q: receive(IncomingQuestion)
  Q->>S: InboundQuestionReceived
  S->>E: registerIncoming + attachSanitized
  S->>W: start(questionId, sanitized)
  W->>R: route(sanitized)
  R-->>W: RoutingDecision{HR}
  W->>E: recordRouting(decision) [emits QuestionRouted]
  E->>Sc: QuestionRouted event
  Sc->>J: score(sanitized, decision)
  J-->>Sc: RouteScore
  Sc->>E: recordRouteScore [emits RouteScored]
  W->>E: recordBranch(HR) [emits RoutingBranched]
  W->>H: runSingleTask(ANSWER, prompt)
  H-->>W: Answer
  W->>E: recordDraft(answer) [emits AnswerDrafted]
  W->>G: check(sanitized, answer)
  G-->>W: GuardrailVerdict{allowed=true}
  W->>E: publish(answer) [emits AnswerPublished, status ANSWERED]
```

The eval-score sequence (steps 7–10) runs concurrently with the workflow's continuation — `RouteEvalScorer` is a Consumer reading the entity's event stream, independent of `HelpWorkflow`. Both writes target the same `QuestionEntity`; commands are idempotent on `questionId`.

## State machine — `QuestionEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: QuestionSanitized
  SANITIZED --> ROUTED: QuestionRouted
  ROUTED --> ROUTED_HR: topic = HR
  ROUTED --> ROUTED_IT: topic = IT
  ROUTED --> ROUTED_POLICY: topic = POLICY
  ROUTED --> ESCALATED: topic = UNCLEAR
  ROUTED_HR --> ANSWER_DRAFTED: AnswerDrafted
  ROUTED_IT --> ANSWER_DRAFTED: AnswerDrafted
  ROUTED_POLICY --> ANSWER_DRAFTED: AnswerDrafted
  ANSWER_DRAFTED --> ANSWERED: guardrail.allowed
  ANSWER_DRAFTED --> BLOCKED: guardrail.rejected
  BLOCKED --> ANSWERED: HR-team unblock
  ANSWERED --> [*]
  ESCALATED --> [*]
```

The `RouteScored` event does not change `status`; it attaches the eval result. The state machine treats it as a no-op transition (omitted for clarity).

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  QuestionEntity ||--o{ QuestionRegistered : emits
  QuestionEntity ||--o{ QuestionSanitized : emits
  QuestionEntity ||--o{ QuestionRouted : emits
  QuestionEntity ||--o{ RoutingBranched : emits
  QuestionEntity ||--o{ AnswerDrafted : emits
  QuestionEntity ||--o{ GuardrailVerdictAttached : emits
  QuestionEntity ||--o{ AnswerPublished : emits
  QuestionEntity ||--o{ AnswerBlocked : emits
  QuestionEntity ||--o{ QuestionEscalated : emits
  QuestionEntity ||--o{ RouteScored : emits
  QuestionView }o--|| QuestionEntity : projects
  QuestionQueue ||--o{ InboundQuestionReceived : emits
  PiiSanitizer }o--|| QuestionQueue : subscribes
  RouteEvalScorer }o--|| QuestionEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QuestionSimulator` | `application/QuestionSimulator.java` |
| `QuestionQueue` | `application/QuestionQueue.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `TopicRouter` | `application/TopicRouter.java` |
| `HRSpecialist` | `application/HRSpecialist.java` |
| `ITSpecialist` | `application/ITSpecialist.java` |
| `PolicySpecialist` | `application/PolicySpecialist.java` |
| `RouteJudge` | `application/RouteJudge.java` |
| `AnswerGuardrail` | `application/AnswerGuardrail.java` |
| `HelpWorkflow` | `application/HelpWorkflow.java` |
| `QuestionEntity` | `application/QuestionEntity.java` (state in `domain/Question.java`, events in `domain/QuestionEvent.java`) |
| `QuestionView` | `application/QuestionView.java` |
| `RouteEvalScorer` | `application/RouteEvalScorer.java` |
| `HelpdeskEndpoint` | `api/HelpdeskEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/HelpTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `routeStep` 20 s, `guardrailStep` 20 s, `hrStep` / `itStep` / `policyStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the question to `ESCALATED` with the failure reason captured.
- **Idempotency.** Every per-question primitive is keyed by `questionId`: `QuestionEntity` id is `questionId`; `HelpWorkflow` id is `questionId`; agent sessions for `TopicRouter`, `RouteJudge`, and `AnswerGuardrail` use `questionId`. Duplicate sanitize events fold into a single workflow start (workflow start is idempotent per id).
- **Race between eval and workflow.** `RouteEvalScorer` (Consumer) and `HelpWorkflow` both append events to the same `QuestionEntity`. Order is not guaranteed but does not matter: `RouteScored` only mutates `routeScore`, never `status`. The view materialises both events independently.
- **No saga compensation.** The handoff is a single-direction transfer of ownership; once the specialist returns its `Answer`, the workflow either publishes or blocks based on the guardrail verdict. There is no rollback path — a blocked draft sits in `BLOCKED` until an HR-team member unblocks via `POST /api/questions/{id}/unblock`.
- **No HITL on the happy path.** The system only waits for a human when the guardrail blocks; everything else flows through to `ANSWERED` autonomously.
- **Simulator throughput.** `QuestionSimulator` drips one question every 30 s; the system can comfortably process each question end-to-end inside that window with mock or real LLMs.
