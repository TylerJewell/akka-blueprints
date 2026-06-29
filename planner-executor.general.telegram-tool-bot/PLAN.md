# PLAN — telegram-tool-bot

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams render on the generated system's Architecture tab.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Router[RouterAgent]:::agent
  WebLookup[WebLookupAgent]:::agent
  ContactBook[ContactBookAgent]:::agent
  Calendar[CalendarQueryAgent]:::agent
  Notes[NoteSaverAgent]:::agent

  WF[SessionWorkflow]:::wf
  Session[SessionEntity]:::ese
  BotCtrl[BotControlEntity]:::ese
  MQ[MessageQueue]:::ese
  View[SessionView]:::view
  Consumer[MessageConsumer]:::cons
  Sim[MessageSimulator]:::ta
  Stale[StaleSessionMonitor]:::ta
  API[SessionEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit| MQ
  API -->|pause/resume| BotCtrl
  Sim -.->|every 75s| MQ
  MQ -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PARSE_INTENT / DECIDE / COMPOSE_REPLY| Router
  WF -->|WEB_LOOKUP| WebLookup
  WF -->|CONTACT_QUERY| ContactBook
  WF -->|CALENDAR_READ| Calendar
  WF -->|NOTE_SAVE| Notes
  WF -->|emit events| Session
  WF -->|poll| BotCtrl
  Session -.->|projects| View
  API -->|query/SSE| View
  Stale -.->|every 30s| Session
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as SessionEndpoint
  participant Q as MessageQueue
  participant C as MessageConsumer
  participant W as SessionWorkflow
  participant R as RouterAgent
  participant T as Tool Executor (Web/Contacts/Calendar/Notes)
  participant E as SessionEntity
  participant CTL as BotControlEntity
  participant V as SessionView

  U->>API: POST /api/sessions {text}
  API->>Q: append MessageReceived
  API-->>U: 202 {sessionId}
  Q->>C: MessageReceived
  C->>W: start({sessionId, text})
  W->>E: emit SessionCreated (PLANNING)
  W->>R: PARSE_INTENT(text)
  R-->>W: SessionLedger
  W->>E: emit SessionPlanned, status EXECUTING
  loop until Reply | Fail | Pause
    W->>CTL: get pause flag
    CTL-->>W: paused=false
    W->>R: DECIDE(ledgers)
    R-->>W: Continue(ToolDispatch)
    W->>W: guardrail.vet(dispatch)
    W->>T: runSingleTask(subtask)
    T-->>W: ToolResult
    W->>W: SecretScrubber.scrub(content)
    W->>E: emit ToolCallRecorded (ToolEntry)
  end
  W->>R: COMPOSE_REPLY
  R-->>W: BotReply
  W->>E: emit SessionCompleted
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> PLANNING
  PLANNING --> EXECUTING: SessionPlanned
  EXECUTING --> EXECUTING: ToolCallRecorded / ToolCallBlocked / LedgerRevised
  EXECUTING --> COMPLETED: SessionCompleted
  EXECUTING --> FAILED: SessionFailed
  EXECUTING --> PAUSED: SessionPausedOperator
  EXECUTING --> STALE: SessionFailedTimeout
  STALE --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
  PAUSED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionPlanned : emits
  SessionEntity ||--o{ ToolDispatched : emits
  SessionEntity ||--o{ ToolCallBlocked : emits
  SessionEntity ||--o{ ToolCallRecorded : emits
  SessionEntity ||--o{ LedgerRevised : emits
  SessionEntity ||--o{ SessionCompleted : emits
  SessionEntity ||--o{ SessionFailed : emits
  SessionEntity ||--o{ SessionPausedOperator : emits
  SessionEntity ||--o{ SessionFailedTimeout : emits
  SessionView }o--|| SessionEntity : projects
  BotControlEntity ||--o{ PauseRequested : emits
  BotControlEntity ||--o{ PauseCleared : emits
  MessageQueue ||--o{ MessageReceived : emits
  MessageConsumer }o--|| MessageQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RouterAgent` | `application/RouterAgent.java` |
| `WebLookupAgent` | `application/WebLookupAgent.java` |
| `ContactBookAgent` | `application/ContactBookAgent.java` |
| `CalendarQueryAgent` | `application/CalendarQueryAgent.java` |
| `NoteSaverAgent` | `application/NoteSaverAgent.java` |
| `SessionWorkflow` | `application/SessionWorkflow.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `BotControlEntity` | `application/BotControlEntity.java` |
| `MessageQueue` | `application/MessageQueue.java` |
| `SessionView` | `application/SessionView.java` |
| `MessageConsumer` | `application/MessageConsumer.java` |
| `MessageSimulator` | `application/MessageSimulator.java` |
| `StaleSessionMonitor` | `application/StaleSessionMonitor.java` |
| `ToolGuardrail` | `application/ToolGuardrail.java` |
| `SecretScrubber` | `application/SecretScrubber.java` |
| `RouterTasks` | `application/RouterTasks.java` |
| `ToolExecutorTasks` | `application/ToolExecutorTasks.java` |
| `SessionEndpoint` | `api/SessionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `proposeStep` 45 s, `dispatchStep` 120 s (covers any tool call), `decideStep` 45 s, `replyStep` 60 s. Default recovery: `maxRetries(2).failoverTo(SessionWorkflow::error)`.
- **Replan budget:** the router may emit `Replan` at most twice in a row without a `Continue` in between; a third consecutive `Replan` is treated as `Fail`.
- **Failure budget:** the router may emit `Continue` on the same `(tool, subtask)` at most three times; a fourth attempt is treated as `Fail`.
- **Pause poll:** every `checkPauseStep` reads `BotControlEntity.get` synchronously — no caching. An operator pause arriving during a `dispatchStep` lets the in-flight tool call finish; the loop exits at the next `checkPauseStep`.
- **Idempotency:** `SessionEndpoint.submit` uses `(text, chatId)` over a 10 s window to dedupe `POST /api/sessions`.
- **Stale detection:** `StaleSessionMonitor` ticks every 30 s; `SessionFailedTimeout` is non-fatal to other sessions. The workflow's `decideStep` checks the entity's status and exits if it reads `STALE`.
- **Sanitizer determinism:** `SecretScrubber.scrub` is pure; it never inspects external state. The same input always yields the same scrubbed output, keeping `ToolEntry` events deterministic and replayable.
