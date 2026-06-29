# PLAN — cx-handoff-triage

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

  Sim[ConversationSimulator]:::ta
  Queue[ConversationQueue]:::ese
  Sanit[PiiSanitizer]:::cons
  Triage[TriageAgent]:::agent
  Sales[SalesSpecialist]:::autonomous
  Issues[IssuesRepairsSpecialist]:::autonomous
  RGuard[ResponseGuardrail]:::agent
  TGuard[ToolCallGuardrail]:::agent
  WF[ConversationWorkflow]:::wf
  Entity[ConversationEntity]:::ese
  View[ConversationView]:::view
  API[ChatEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| Sanit
  Sanit -->|register + sanitize| Entity
  Sanit -->|start workflow| WF
  WF -->|triage| Triage
  WF -->|RESOLVE task| Sales
  WF -->|RESOLVE task| Issues
  Sales -->|tool call| TGuard
  Issues -->|tool call| TGuard
  WF -->|check draft| RGuard
  WF -->|emit events| Entity
  Entity -.->|projects| View
  API -->|unblock / submit| Entity
  API -->|query / SSE| View
  App -->|static UI + metadata| API
```

Solid arrows = synchronous component calls. Dashed arrows = event subscriptions and scheduler ticks.

## Interaction sequence — J1 (sales happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'actorTextColor':'#ffffff','noteTextColor':'#ffffff','sequenceNumberColor':'#000'
}}}%%
sequenceDiagram
  autonumber
  participant Sim as ConversationSimulator
  participant Q as ConversationQueue
  participant S as PiiSanitizer
  participant E as ConversationEntity
  participant W as ConversationWorkflow
  participant T as TriageAgent
  participant Sp as SalesSpecialist
  participant TG as ToolCallGuardrail
  participant RG as ResponseGuardrail

  Sim->>Q: receive(InboundMessage)
  Q->>S: InboundMessageReceived
  S->>E: registerIncoming + attachSanitized
  S->>W: start(conversationId, sanitized)
  W->>T: triage(sanitized)
  T-->>W: TriageDecision{SALES}
  W->>E: recordTriage(decision) [emits ConversationTriaged]
  W->>E: recordRouting(SALES) [emits ConversationRouted]
  W->>Sp: runSingleTask(RESOLVE, prompt)
  Sp->>TG: check(ToolCallRequest{placeOrder, ...})
  TG-->>Sp: ToolCallVerdict{allowed=true}
  Sp-->>W: SpecialistResponse
  W->>E: recordDraft(response) [emits ResponseDrafted]
  W->>RG: check(sanitized, response)
  RG-->>W: GuardrailVerdict{allowed=true}
  W->>E: publish(response) [emits ResponsePublished, status RESOLVED]
```

The tool-call guardrail (steps 10–11) runs inline within the specialist's task loop, before the tool executes. If the verdict is `allowed=false`, the tool call is cancelled, `ToolCallBlocked` is emitted on `ConversationEntity`, and the specialist receives a policy-rejection signal within the same iteration.

## State machine — `ConversationEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: ConversationSanitized
  SANITIZED --> TRIAGED: ConversationTriaged
  TRIAGED --> ROUTED_SALES: category = SALES
  TRIAGED --> ROUTED_ISSUES_REPAIRS: category = ISSUES_REPAIRS
  TRIAGED --> ESCALATED: category = UNCLEAR
  ROUTED_SALES --> RESPONSE_DRAFTED: ResponseDrafted
  ROUTED_ISSUES_REPAIRS --> RESPONSE_DRAFTED: ResponseDrafted
  ROUTED_SALES --> TOOL_CALL_BLOCKED: ToolCallBlocked
  ROUTED_ISSUES_REPAIRS --> TOOL_CALL_BLOCKED: ToolCallBlocked
  TOOL_CALL_BLOCKED --> RESPONSE_DRAFTED: specialist pivots
  RESPONSE_DRAFTED --> RESOLVED: guardrail.allowed
  RESPONSE_DRAFTED --> BLOCKED: guardrail.rejected
  BLOCKED --> RESOLVED: operator unblock
  RESOLVED --> [*]
  ESCALATED --> [*]
```

`TOOL_CALL_BLOCKED` is a transient annotation state — after the specialist receives the rejection signal it continues the task loop and eventually still produces a `SpecialistResponse` that moves the conversation to `RESPONSE_DRAFTED`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  ConversationEntity ||--o{ ConversationRegistered : emits
  ConversationEntity ||--o{ ConversationSanitized : emits
  ConversationEntity ||--o{ ConversationTriaged : emits
  ConversationEntity ||--o{ ConversationRouted : emits
  ConversationEntity ||--o{ ResponseDrafted : emits
  ConversationEntity ||--o{ ToolCallBlocked : emits
  ConversationEntity ||--o{ GuardrailVerdictAttached : emits
  ConversationEntity ||--o{ ResponsePublished : emits
  ConversationEntity ||--o{ ResponseBlocked : emits
  ConversationEntity ||--o{ ConversationEscalated : emits
  ConversationView }o--|| ConversationEntity : projects
  ConversationQueue ||--o{ InboundMessageReceived : emits
  PiiSanitizer }o--|| ConversationQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ConversationSimulator` | `application/ConversationSimulator.java` |
| `ConversationQueue` | `application/ConversationQueue.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `TriageAgent` | `application/TriageAgent.java` |
| `SalesSpecialist` | `application/SalesSpecialist.java` |
| `IssuesRepairsSpecialist` | `application/IssuesRepairsSpecialist.java` |
| `ResponseGuardrail` | `application/ResponseGuardrail.java` |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `ConversationWorkflow` | `application/ConversationWorkflow.java` |
| `ConversationEntity` | `application/ConversationEntity.java` (state in `domain/Conversation.java`, events in `domain/ConversationEvent.java`) |
| `ConversationView` | `application/ConversationView.java` |
| `ChatEndpoint` | `api/ChatEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/ChatTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `triageStep` 20 s, `guardrailStep` 20 s, `salesStep` / `issuesStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the conversation to `ESCALATED` with the failure reason captured.
- **Idempotency.** Every per-conversation primitive is keyed by `conversationId`: `ConversationEntity` id is `conversationId`; `ConversationWorkflow` id is `conversationId`; agent sessions for `TriageAgent`, `ResponseGuardrail`, and `ToolCallGuardrail` use `conversationId`. Duplicate sanitize events fold into a single workflow start.
- **Tool-call guardrail ordering.** The `ToolCallGuardrail` check happens inline within the specialist's autonomous task loop, before any tool side-effect occurs. The specialist may retry with a different tool or parameter set within its `maxIterationsPerTask` budget.
- **No saga compensation.** A blocked draft sits in `BLOCKED` until an operator unblocks via `POST /api/conversations/{id}/unblock`. There is no automatic retry path after a `ResponseBlocked` event.
- **No HITL on the happy path.** The system only waits for a human when the response guardrail blocks; everything else flows through to `RESOLVED` autonomously. This is the distinction from a human-in-loop-gate pattern.
- **Simulator throughput.** `ConversationSimulator` drips one conversation every 30 s; the system can comfortably process each one end-to-end inside that window with mock or real LLMs.
