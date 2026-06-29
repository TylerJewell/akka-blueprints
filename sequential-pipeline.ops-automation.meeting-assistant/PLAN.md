# Implementation Plan — `meeting-assistant`

The architecture this sample resolves to once `SPEC.md` runs through `/akka:specify` → `/akka:plan`.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1c1c1c','primaryBorderColor':'#e6c200','primaryTextColor':'#ffffff','lineColor':'#e6c200','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans, sans-serif'}}}%%
flowchart TB
  Sim[NotesSimulator\nTimedAction] -.->|every 30s| Queue[InboundNotesQueue\nEventSourcedEntity]
  Endpoint[MeetingEndpoint\nHttpEndpoint] -->|enqueueNotes| Queue
  Queue -.->|InboundNotesQueued| Consumer[NotesConsumer\nConsumer]
  Consumer -->|start| WF[PipelineWorkflow\nWorkflow]
  WF -->|sanitizeStep| Entity[MeetingEntity\nEventSourcedEntity]
  WF -->|extractStep| Agent[ActionExtractorAgent\nAutonomousAgent]
  Agent --> Entity
  WF -->|dispatchStep| Sim2[SimEndpoint\nHttpEndpoint]
  WF --> Entity
  Entity -.->|events| View[MeetingsView\nView]
  View --> Endpoint
  Retry[DispatchRetryMonitor\nTimedAction] -.->|every 30s| View
  Retry --> WF
  App[AppEndpoint\nHttpEndpoint] --> Static[static-resources/index.html]
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions or scheduled ticks.

## Interaction sequence

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1c1c1c','primaryBorderColor':'#e6c200','primaryTextColor':'#ffffff','lineColor':'#e6c200','actorTextColor':'#ffffff','fontFamily':'Instrument Sans, sans-serif'}}}%%
sequenceDiagram
  participant U as User
  participant E as MeetingEndpoint
  participant Q as InboundNotesQueue
  participant C as NotesConsumer
  participant W as PipelineWorkflow
  participant A as ActionExtractorAgent
  participant M as MeetingEntity
  participant S as SimEndpoint
  U->>E: POST /api/meetings {title, notes}
  E->>Q: enqueueNotes
  Q-->>C: InboundNotesQueued
  C->>W: start(meetingId, notes)
  W->>M: recordSanitized (redacted text)
  Note over W: PII redaction before any model call
  W->>A: runSingleTask(EXTRACT, redacted)
  A-->>W: ActionPlan
  W->>M: recordExtraction
  Note over W: before-tool-call guardrail per write
  W->>S: POST /sim/trello/cards
  W->>S: POST /sim/slack/messages
  W->>M: recordDispatch
```

## State machine

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1c1c1c','primaryBorderColor':'#e6c200','primaryTextColor':'#ffffff','lineColor':'#e6c200','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc','fontFamily':'Instrument Sans, sans-serif'}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: NotesSanitized
  SANITIZED --> EXTRACTED: ActionsExtracted
  EXTRACTED --> DISPATCHED: ActionsDispatched
  EXTRACTED --> FAILED: DispatchFailed (guardrail refusal)
  DISPATCHED --> [*]
  FAILED --> [*]
```

State labels and transition labels carry explicit colour via the Lesson 24 CSS overrides in the generated `index.html`:

```css
.diagram-card .mermaid .stateLabel,
.diagram-card .mermaid g.statediagram-state .label,
.diagram-card .mermaid g.statediagram-state .label *,
.diagram-card .mermaid g.statediagram-state text { color:#ffffff !important; fill:#ffffff !important; }
.diagram-card .mermaid .edgeLabel foreignObject { overflow:visible !important; }
```

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1c1c1c','primaryBorderColor':'#e6c200','primaryTextColor':'#ffffff','lineColor':'#e6c200','fontFamily':'Instrument Sans, sans-serif'}}}%%
erDiagram
  MEETING ||--o{ ACTION_ITEM : contains
  MEETING {
    string id
    string status
    string summary
    int redactionCount
  }
  ACTION_ITEM {
    string title
    string assignee
    string dueHint
    string trelloCardUrl
  }
  INBOUND_NOTES_QUEUE {
    string id
    string title
    string rawNotes
  }
```

## Component table

| Component | Kind | File | Purpose |
|---|---|---|---|
| `ActionExtractorAgent` | AutonomousAgent | `application/ActionExtractorAgent.java` | Redacted transcript → `ActionPlan`. |
| `ExtractionTasks` | task definitions | `application/ExtractionTasks.java` | `Task<ActionPlan> EXTRACT`. |
| `MeetingEntity` | EventSourcedEntity | `application/MeetingEntity.java` | Per-meeting lifecycle. |
| `InboundNotesQueue` | EventSourcedEntity | `application/InboundNotesQueue.java` | Inbound submissions. |
| `PipelineWorkflow` | Workflow | `application/PipelineWorkflow.java` | `sanitizeStep` → `extractStep` → `dispatchStep`. |
| `MeetingsView` | View | `application/MeetingsView.java` | Row type `Meeting`; one query + SSE stream. |
| `NotesConsumer` | Consumer | `application/NotesConsumer.java` | Starts a workflow per submission. |
| `NotesSimulator` | TimedAction | `application/NotesSimulator.java` | Drips canned notes every 30s. |
| `DispatchRetryMonitor` | TimedAction | `application/DispatchRetryMonitor.java` | Re-drives stuck `EXTRACTED` meetings. |
| `PiiRedactor` | helper | `application/PiiRedactor.java` | Deterministic PII redaction. |
| `TrelloClient` | helper | `application/TrelloClient.java` | Real Trello or in-process sim. |
| `SlackClient` | helper | `application/SlackClient.java` | Real Slack or in-process sim. |
| `MeetingEndpoint` | HttpEndpoint | `api/MeetingEndpoint.java` | `/api/*` + SSE + metadata. |
| `SimEndpoint` | HttpEndpoint | `api/SimEndpoint.java` | `/sim/trello/cards`, `/sim/slack/messages`. |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` | Serves the UI. |
| `Bootstrap` | service-setup | `Bootstrap.java` | Schedules the TimedActions. |

Component count: **3 http-endpoint · 2 timed-action · 1 view · 1 workflow · 1 service-setup · 1 autonomous-agent · 1 consumer · 2 event-sourced-entity**.

## Concurrency notes

- **Step timeouts.** `extractStep` calls the agent — set `stepTimeout(extractStep, 60s)` (Lesson 4). `dispatchStep` makes HTTP writes — `stepTimeout(dispatchStep, 30s)`. `defaultStepRecovery(maxRetries(2))`.
- **Idempotency.** Each dispatch carries the `meetingId` plus action index as an idempotency key so a workflow retry does not create duplicate Trello cards. The `SimEndpoint` dedupes on that key.
- **Compensation.** If the Slack post fails after Trello cards were created, `dispatchStep` records the partial result and `DispatchFailed`; the retry monitor re-drives only the missing writes, keyed by idempotency key — no card is recreated.
- **View indexing.** `MeetingsView` has no `WHERE status` clause; status filtering is client-side because Akka cannot auto-index enum columns (Lesson 2).
