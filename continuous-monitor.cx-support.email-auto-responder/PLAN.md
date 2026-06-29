# Implementation Plan — `email-auto-responder`

The architecture this blueprint resolves to once [`SPEC.md`](./SPEC.md) is run through `/akka:specify` → `/akka:plan`.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1a1a1a','primaryBorderColor':'#e8b923','primaryTextColor':'#ffffff','lineColor':'#e8b923','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans, sans-serif'}}}%%
flowchart TB
  InboxMonitor[InboxMonitor\nTimedAction]
  InboxQueue[InboxQueue\nEventSourcedEntity]
  Consumer[InboundEmailConsumer\nConsumer]
  Workflow[ResponderWorkflow\nWorkflow]
  Triage[TriageAgent\nAgent]
  Responder[ResponderAgent\nAutonomousAgent]
  EmailEntity[EmailEntity\nEventSourcedEntity]
  EmailsView[EmailsView\nView]
  StuckMonitor[StuckReviewMonitor\nTimedAction]
  RespEndpoint[ResponderEndpoint\nHttpEndpoint]
  AppEndpoint[AppEndpoint\nHttpEndpoint]

  InboxMonitor -. drips every 30s .-> InboxQueue
  InboxQueue -. EmailQueued .-> Consumer
  Consumer --> Workflow
  Workflow --> Triage
  Workflow --> Responder
  Workflow --> EmailEntity
  EmailEntity -. events .-> EmailsView
  StuckMonitor -. every 30s .-> EmailsView
  StuckMonitor --> EmailEntity
  RespEndpoint --> EmailEntity
  RespEndpoint --> EmailsView
  RespEndpoint --> InboxQueue
  AppEndpoint --> RespEndpoint
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions; dotted arrows are scheduled ticks.

## Interaction sequence

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1a1a1a','primaryBorderColor':'#e8b923','primaryTextColor':'#ffffff','lineColor':'#e8b923','fontFamily':'Instrument Sans, sans-serif'}}}%%
sequenceDiagram
  participant M as InboxMonitor
  participant Q as InboxQueue
  participant C as InboundEmailConsumer
  participant W as ResponderWorkflow
  participant T as TriageAgent
  participant R as ResponderAgent
  participant E as EmailEntity
  participant H as Human (UI)

  M->>Q: receive(email)
  Q-->>C: EmailQueued
  C->>E: ingest(email)
  C->>W: start(emailId)
  W->>E: recordSanitized(maskedBody)
  W->>T: classify(sanitizedBody)
  W->>R: runSingleTask(DRAFT_REPLY)
  W->>E: recordDraft(reply, sensitive)
  Note over W,H: sensitive → pause in AWAITING_REVIEW
  H->>E: approve
  W->>E: recordSent(messageId)
```

## State machine

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1a1a1a','primaryBorderColor':'#e8b923','primaryTextColor':'#ffffff','lineColor':'#e8b923','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc','fontFamily':'Instrument Sans, sans-serif'}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: sanitizeStep
  SANITIZED --> DRAFTED: draftStep
  DRAFTED --> AWAITING_REVIEW: sensitive
  DRAFTED --> SENT: routine auto-send
  AWAITING_REVIEW --> APPROVED: human approve
  AWAITING_REVIEW --> REJECTED: human reject
  AWAITING_REVIEW --> ESCALATED: timeout
  APPROVED --> SENT: sendStep
  SENT --> [*]
  REJECTED --> [*]
  ESCALATED --> [*]
```

State-label and transition-label colours require the Lesson 24 CSS overrides in the generated UI; the theme variables above reduce flicker before the CSS lands.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1a1a1a','primaryBorderColor':'#e8b923','primaryTextColor':'#ffffff','lineColor':'#e8b923','fontFamily':'Instrument Sans, sans-serif'}}}%%
erDiagram
  INBOX_QUEUE ||--o{ EMAIL : enqueues
  EMAIL ||--o{ EMAIL_EVENT : emits
  EMAIL ||--|| EMAILS_VIEW : projects
  EMAIL {
    string id
    string fromAddress
    string subject
    enum status
    optional sanitizedBody
    optional draftBody
    optional sensitive
    optional sentMessageId
  }
  EMAIL_EVENT {
    string type
    instant timestamp
  }
```

## Component table

| Component | Kind | File |
|---|---|---|
| `ResponderAgent` | AutonomousAgent | `application/ResponderAgent.java` |
| `ResponderTasks` | task definitions | `application/ResponderTasks.java` |
| `TriageAgent` | Agent | `application/TriageAgent.java` |
| `ResponderWorkflow` | Workflow | `application/ResponderWorkflow.java` |
| `EmailEntity` | EventSourcedEntity | `application/EmailEntity.java` |
| `InboxQueue` | EventSourcedEntity | `application/InboxQueue.java` |
| `EmailsView` | View | `application/EmailsView.java` |
| `InboundEmailConsumer` | Consumer | `application/InboundEmailConsumer.java` |
| `InboxMonitor` | TimedAction | `application/InboxMonitor.java` |
| `StuckReviewMonitor` | TimedAction | `application/StuckReviewMonitor.java` |
| `ResponderEndpoint` | HttpEndpoint | `api/ResponderEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |
| `Bootstrap` | service-setup | `Bootstrap.java` |
| `Email`, `EmailStatus`, `EmailEvent` | domain | `domain/*.java` |

Akka component count: **2 http-endpoint · 2 timed-action · 1 view · 1 workflow · 1 service-setup · 1 autonomous-agent · 1 agent · 1 consumer · 2 event-sourced-entity**.

## Concurrency notes

- `ResponderWorkflow.settings()` sets `stepTimeout` 60s on `draftStep` and `sendStep` (LLM-calling and side-effecting), 10s on `sanitizeStep` and `reviewGateStep` (Lesson 4). `WorkflowSettings` is nested in `Workflow` — no import.
- `reviewGateStep` self-schedules a 5s resume timer while sensitive emails sit in `AWAITING_REVIEW`; it ends on `APPROVED`, `REJECTED`, or `ESCALATED`.
- Idempotency: the workflow keys on the email id; `InboundEmailConsumer` ingests the `EmailEntity` before starting the workflow so a redelivered `EmailQueued` re-uses the same entity.
- Compensation: there is no real external mail server; the send action is in-process, so no saga rollback is needed. The guardrail makes the send the only side-effecting step and gates it on status.
- `EmailsView` has one query with no enum `WHERE` clause (Lesson 2); status filtering happens client-side in the endpoint and `StuckReviewMonitor`.
