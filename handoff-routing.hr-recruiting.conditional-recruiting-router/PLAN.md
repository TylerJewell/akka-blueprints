# PLAN — conditional-recruiting-router

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

  Sim[EmailSimulator]:::ta
  Inbox[InboxQueue]:::ese
  Sanit[CandidateSanitizer]:::cons
  Classifier[EmailClassifierAgent]:::agent
  Info[InfoRequester]:::autonomous
  Organizer[InterviewOrganizer]:::autonomous
  Judge[RoutingJudge]:::agent
  Guard[ToolCallGuardrail]:::agent
  WF[RecruitingWorkflow]:::wf
  Entity[ApplicationEntity]:::ese
  View[ApplicationView]:::view
  Scorer[RoutingEvalScorer]:::cons
  API[RecruitingEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Inbox
  Inbox -.->|subscribes| Sanit
  Sanit -->|register + sanitize| Entity
  Sanit -->|start workflow| WF
  WF -->|classify| Classifier
  WF -->|REPLY task| Info
  WF -->|SCHEDULE task| Organizer
  WF -->|check tool call| Guard
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

## Interaction sequence — J2 (interview-request happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'actorTextColor':'#ffffff','noteTextColor':'#ffffff','sequenceNumberColor':'#000'
}}}%%
sequenceDiagram
  autonumber
  participant Sim as EmailSimulator
  participant Q as InboxQueue
  participant S as CandidateSanitizer
  participant E as ApplicationEntity
  participant W as RecruitingWorkflow
  participant C as EmailClassifierAgent
  participant O as InterviewOrganizer
  participant G as ToolCallGuardrail
  participant Sc as RoutingEvalScorer
  participant J as RoutingJudge

  Sim->>Q: receive(InboundEmail)
  Q->>S: InboundEmailReceived
  S->>E: registerIncoming + attachSanitized
  S->>W: start(applicationId, sanitized)
  W->>C: classify(sanitized)
  C-->>W: RoutingDecision{INTERVIEW_REQUEST}
  W->>E: recordRouting(decision) [emits RoutingDecided]
  E->>Sc: RoutingDecided event
  Sc->>J: score(sanitized, decision)
  J-->>Sc: RoutingScore
  Sc->>E: recordRoutingScore [emits RoutingScored]
  W->>E: recordApplicationRouted(INTERVIEW_REQUEST)
  W->>O: runSingleTask(SCHEDULE, prompt)
  O->>G: check(schedule_slot, args)
  G-->>O: ToolCallVerdict{allowed=true}
  O-->>W: CalendarConfirmation
  W->>E: recordSlotProposed + complete [emits SlotProposed, ApplicationCompleted]
```

The eval-event sequence (steps 7–10) runs concurrently with the workflow's continuation — `RoutingEvalScorer` is a Consumer reading the entity's event stream, independent of `RecruitingWorkflow`. Both writes target the same `ApplicationEntity`; commands are idempotent on `applicationId`.

## State machine — `ApplicationEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: ApplicationSanitized
  SANITIZED --> CLASSIFIED: RoutingDecided
  CLASSIFIED --> ROUTED_INFO: route = INFO_REQUEST
  CLASSIFIED --> ROUTED_INTERVIEW: route = INTERVIEW_REQUEST
  CLASSIFIED --> UNROUTABLE_CLOSED: route = UNROUTABLE
  ROUTED_INFO --> REPLY_DRAFTED: ReplyDrafted
  ROUTED_INTERVIEW --> SLOT_PROPOSED: SlotProposed
  ROUTED_INTERVIEW --> TOOL_BLOCKED: ToolCallBlocked
  REPLY_DRAFTED --> COMPLETED: ApplicationCompleted
  SLOT_PROPOSED --> COMPLETED: ApplicationCompleted
  TOOL_BLOCKED --> COMPLETED: recruiter unblock
  COMPLETED --> [*]
  UNROUTABLE_CLOSED --> [*]
```

The `RoutingScored` event does not change `status`; it attaches the eval result. The state machine treats it as a no-op transition (omitted for clarity).

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  ApplicationEntity ||--o{ ApplicationRegistered : emits
  ApplicationEntity ||--o{ ApplicationSanitized : emits
  ApplicationEntity ||--o{ RoutingDecided : emits
  ApplicationEntity ||--o{ ApplicationRouted : emits
  ApplicationEntity ||--o{ ReplyDrafted : emits
  ApplicationEntity ||--o{ SlotProposed : emits
  ApplicationEntity ||--o{ ToolCallBlocked : emits
  ApplicationEntity ||--o{ ApplicationCompleted : emits
  ApplicationEntity ||--o{ ApplicationClosed : emits
  ApplicationEntity ||--o{ RoutingScored : emits
  ApplicationView }o--|| ApplicationEntity : projects
  InboxQueue ||--o{ InboundEmailReceived : emits
  CandidateSanitizer }o--|| InboxQueue : subscribes
  RoutingEvalScorer }o--|| ApplicationEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `EmailSimulator` | `application/EmailSimulator.java` |
| `InboxQueue` | `application/InboxQueue.java` |
| `CandidateSanitizer` | `application/CandidateSanitizer.java` |
| `EmailClassifierAgent` | `application/EmailClassifierAgent.java` |
| `InfoRequester` | `application/InfoRequester.java` |
| `InterviewOrganizer` | `application/InterviewOrganizer.java` |
| `RoutingJudge` | `application/RoutingJudge.java` |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `RecruitingWorkflow` | `application/RecruitingWorkflow.java` |
| `ApplicationEntity` | `application/ApplicationEntity.java` (state in `domain/Application.java`, events in `domain/ApplicationEvent.java`) |
| `ApplicationView` | `application/ApplicationView.java` |
| `RoutingEvalScorer` | `application/RoutingEvalScorer.java` |
| `RecruitingEndpoint` | `api/RecruitingEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/RecruitingTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `classifyStep` 20 s, `toolGuardrailStep` 20 s, `infoStep` / `scheduleStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the application to `UNROUTABLE_CLOSED` with the failure reason captured.
- **Idempotency.** Every per-application primitive is keyed by `applicationId`: `ApplicationEntity` id is `applicationId`; `RecruitingWorkflow` id is `applicationId`; agent sessions for `EmailClassifierAgent`, `RoutingJudge`, and `ToolCallGuardrail` use `applicationId`. Duplicate sanitize events fold into a single workflow start (workflow start is idempotent per id).
- **Race between eval and workflow.** `RoutingEvalScorer` (Consumer) and `RecruitingWorkflow` both append events to the same `ApplicationEntity`. Order is not guaranteed but does not matter: `RoutingScored` only mutates `routingScore`, never `status`. The view materialises both events independently.
- **No saga compensation.** The handoff is a single-direction transfer of ownership. A tool-blocked application sits in `TOOL_BLOCKED` until a recruiter unblocks via `POST /api/applications/{id}/unblock`.
- **No HITL on the happy path.** The system only waits for a human when the guardrail blocks a tool call; everything else flows through to `COMPLETED` autonomously.
- **Simulator throughput.** `EmailSimulator` drips one email every 30 s; the service can process each application end-to-end inside that window with mock or real LLMs.
