# PLAN — discord-router

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

  Sim[MessageSimulator]:::ta
  Queue[MessageQueue]:::ese
  Sanit[PiiSanitizer]:::cons
  Classifier[ClassifierAgent]:::agent
  Community[CommunitySpecialist]:::autonomous
  Tech[TechnicalSpecialist]:::autonomous
  Judge[RoutingJudge]:::agent
  Guard[ReplyGuardrail]:::agent
  WF[DiscordWorkflow]:::wf
  Entity[MessageEntity]:::ese
  View[MessageView]:::view
  Scorer[RoutingEvalScorer]:::cons
  API[DiscordEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| Sanit
  Sanit -->|register + sanitize| Entity
  Sanit -->|start workflow| WF
  WF -->|classify| Classifier
  WF -->|REPLY task| Community
  WF -->|REPLY task| Tech
  WF -->|check draft| Guard
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

## Interaction sequence — J1 (community happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'actorTextColor':'#ffffff','noteTextColor':'#ffffff','sequenceNumberColor':'#000'
}}}%%
sequenceDiagram
  autonumber
  participant Sim as MessageSimulator
  participant Q as MessageQueue
  participant S as PiiSanitizer
  participant E as MessageEntity
  participant W as DiscordWorkflow
  participant C as ClassifierAgent
  participant Cs as CommunitySpecialist
  participant G as ReplyGuardrail
  participant Sc as RoutingEvalScorer
  participant J as RoutingJudge

  Sim->>Q: receive(IncomingMessage)
  Q->>S: InboundMessageReceived
  S->>E: registerIncoming + attachSanitized
  S->>W: start(messageId, sanitized)
  W->>C: classify(sanitized)
  C-->>W: RoutingDecision{COMMUNITY}
  W->>E: recordClassification(decision) [emits MessageClassified]
  E->>Sc: MessageClassified event
  Sc->>J: score(sanitized, decision)
  J-->>Sc: RoutingScore
  Sc->>E: recordRoutingScore [emits RoutingScored]
  W->>E: recordRouting(COMMUNITY) [emits MessageRouted]
  W->>Cs: runSingleTask(REPLY, prompt)
  Cs-->>W: BotReply
  W->>E: recordDraft(reply) [emits ReplyDrafted]
  W->>G: check(sanitized, reply)
  G-->>W: GuardrailVerdict{allowed=true}
  W->>E: publish(reply) [emits ReplyPublished, status PUBLISHED]
```

The eval-event sequence (steps 8–11) runs concurrently with the workflow's continuation — `RoutingEvalScorer` is a Consumer reading the entity's event stream, independent of `DiscordWorkflow`. Both write to the same `MessageEntity`; the entity's commands are idempotent on `messageId`.

## State machine — `MessageEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: MessageSanitized
  SANITIZED --> CLASSIFIED: MessageClassified
  CLASSIFIED --> ROUTED_COMMUNITY: category = COMMUNITY
  CLASSIFIED --> ROUTED_TECHNICAL: category = TECHNICAL
  CLASSIFIED --> ESCALATED: category = UNCLEAR
  ROUTED_COMMUNITY --> REPLY_DRAFTED: ReplyDrafted
  ROUTED_TECHNICAL --> REPLY_DRAFTED: ReplyDrafted
  REPLY_DRAFTED --> PUBLISHED: guardrail.allowed
  REPLY_DRAFTED --> BLOCKED: guardrail.rejected
  BLOCKED --> PUBLISHED: moderator unblock
  PUBLISHED --> [*]
  ESCALATED --> [*]
```

The `RoutingScored` event does not change `status`; it attaches the eval result as metadata. The state machine therefore treats it as a no-op transition (omitted from the diagram for clarity).

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  MessageEntity ||--o{ MessageRegistered : emits
  MessageEntity ||--o{ MessageSanitized : emits
  MessageEntity ||--o{ MessageClassified : emits
  MessageEntity ||--o{ MessageRouted : emits
  MessageEntity ||--o{ ReplyDrafted : emits
  MessageEntity ||--o{ GuardrailVerdictAttached : emits
  MessageEntity ||--o{ ReplyPublished : emits
  MessageEntity ||--o{ ReplyBlocked : emits
  MessageEntity ||--o{ MessageEscalated : emits
  MessageEntity ||--o{ RoutingScored : emits
  MessageView }o--|| MessageEntity : projects
  MessageQueue ||--o{ InboundMessageReceived : emits
  PiiSanitizer }o--|| MessageQueue : subscribes
  RoutingEvalScorer }o--|| MessageEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `MessageSimulator` | `application/MessageSimulator.java` |
| `MessageQueue` | `application/MessageQueue.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `ClassifierAgent` | `application/ClassifierAgent.java` |
| `CommunitySpecialist` | `application/CommunitySpecialist.java` |
| `TechnicalSpecialist` | `application/TechnicalSpecialist.java` |
| `RoutingJudge` | `application/RoutingJudge.java` |
| `ReplyGuardrail` | `application/ReplyGuardrail.java` |
| `DiscordWorkflow` | `application/DiscordWorkflow.java` |
| `MessageEntity` | `application/MessageEntity.java` (state in `domain/Message.java`, events in `domain/MessageEvent.java`) |
| `MessageView` | `application/MessageView.java` |
| `RoutingEvalScorer` | `application/RoutingEvalScorer.java` |
| `DiscordEndpoint` | `api/DiscordEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/DiscordTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `classifyStep` 20 s, `guardrailStep` 20 s, `communityStep` / `technicalStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the message to `ESCALATED` with the failure reason captured.
- **Idempotency.** Every per-message primitive is keyed by `messageId`: `MessageEntity` id is `messageId`; `DiscordWorkflow` id is `messageId`; agent sessions for `ClassifierAgent`, `RoutingJudge`, and `ReplyGuardrail` use `messageId`. Duplicate sanitize events fold into a single workflow start (workflow start is idempotent per id).
- **Race between eval and workflow.** `RoutingEvalScorer` (Consumer) and `DiscordWorkflow` both append events to the same `MessageEntity`. Order is not guaranteed but does not matter: `RoutingScored` only mutates `routingScore`, never `status`. The view materialises both events independently.
- **No saga compensation.** The handoff is a single-direction transfer of ownership; once the specialist returns its `BotReply`, the workflow either publishes or blocks based on the guardrail verdict. A blocked draft sits in `BLOCKED` until a moderator unblocks via `POST /api/messages/{id}/unblock`.
- **No HITL on the happy path.** The system only waits for a human when the guardrail blocks; everything else flows through to `PUBLISHED` autonomously.
- **Simulator throughput.** `MessageSimulator` drips one message every 30 s; the service can process each message end-to-end inside that window with mock or real LLMs.
