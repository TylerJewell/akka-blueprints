# Implementation Plan — Lead Scoring Strategy

The architecture `SPEC.md` resolves to once run through `/akka:specify` → `/akka:plan`. Diagrams render on the Architecture tab; they carry the Akka theme variables plus the CSS overrides for state-diagram labels and edge-label `foreignObject` overflow (Lesson 24).

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#F5C518','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  SIM[LeadSimulator\nTimedAction]
  Q[InboundLeadQueue\nEventSourcedEntity]
  CON[InboundLeadConsumer\nConsumer]
  WF[LeadStrategyWorkflow\nWorkflow]
  AI[IntakeAgent\nAutonomousAgent]
  AR[ResearchAgent\nAutonomousAgent]
  AS[ScoringAgent\nAutonomousAgent]
  AT[StrategyAgent\nAutonomousAgent]
  LE[LeadEntity\nEventSourcedEntity]
  EV[ScoreEvalConsumer\nConsumer]
  V[LeadsView\nView]
  EP[LeadEndpoint\nHttpEndpoint]
  APP[AppEndpoint\nHttpEndpoint]

  SIM -->|enqueueLead| Q
  EP -->|enqueueLead| Q
  Q -.InboundLeadQueued.-> CON
  CON -->|start| WF
  WF -->|runSingleTask| AI
  WF -->|runSingleTask| AR
  WF -->|runSingleTask| AS
  WF -->|runSingleTask| AT
  WF -->|commands| LE
  LE -.LeadScored.-> EV
  EV -->|recordEvaluation| LE
  LE -.events.-> V
  EP -->|queries/SSE| V
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions.

## Interaction sequence

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#F5C518','actorTextColor':'#ffffff','noteTextColor':'#111111','fontFamily':'Instrument Sans'}}}%%
sequenceDiagram
  participant U as User/Simulator
  participant Q as InboundLeadQueue
  participant W as LeadStrategyWorkflow
  participant A as Agents
  participant L as LeadEntity
  participant E as ScoreEvalConsumer
  U->>Q: enqueueLead(rawSource)
  Q-->>W: start(leadId)
  W->>A: intake (PII sanitized first)
  A-->>W: IntakeProfile
  W->>L: recordIntake (INTAKE)
  W->>A: research
  A-->>W: ResearchBrief
  W->>L: recordResearch (RESEARCHED)
  W->>A: score
  A-->>W: LeadScore
  W->>L: recordScore (SCORED)
  L-->>E: LeadScored
  E->>L: recordEvaluation (eval result)
  W->>A: strategy
  A-->>W: EngagementStrategy
  W->>L: recordStrategy (COMPLETE)
```

## State machine

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#F5C518','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc','fontFamily':'Instrument Sans'}}}%%
stateDiagram-v2
  [*] --> NEW
  NEW --> INTAKE: recordIntake
  INTAKE --> RESEARCHED: recordResearch
  RESEARCHED --> SCORED: recordScore
  SCORED --> COMPLETE: recordStrategy
  COMPLETE --> [*]
```

`recordEvaluation` fires on the `SCORED` state from `ScoreEvalConsumer` and sets the eval fields without changing the status. State labels need the CSS overrides from Lesson 24 — the `g.statediagram-state .label` path does not inherit `primaryTextColor`, and edge labels clip without `overflow:visible` on their `foreignObject`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#F5C518','fontFamily':'Instrument Sans'}}}%%
erDiagram
  INBOUND_LEAD_QUEUE ||--o{ LEAD : starts_workflow
  LEAD ||--o{ LEAD_EVENT : emits
  LEAD ||--|| LEADS_VIEW : projects
  LEAD_EVENT {
    string type
    string leadId
    instant timestamp
  }
  LEAD {
    string id
    string status
    int score
    int evalScore
  }
  LEADS_VIEW {
    string id
    string status
    int score
  }
```

## Component table

| Component | Kind | File |
|---|---|---|
| `IntakeAgent` | AutonomousAgent | `application/IntakeAgent.java` |
| `ResearchAgent` | AutonomousAgent | `application/ResearchAgent.java` |
| `ScoringAgent` | AutonomousAgent | `application/ScoringAgent.java` |
| `StrategyAgent` | AutonomousAgent | `application/StrategyAgent.java` |
| `LeadStrategyTasks` | task definitions | `application/LeadStrategyTasks.java` |
| `PiiSanitizer` | helper | `application/PiiSanitizer.java` |
| `LeadStrategyWorkflow` | Workflow | `application/LeadStrategyWorkflow.java` |
| `LeadEntity` | EventSourcedEntity | `application/LeadEntity.java` |
| `InboundLeadQueue` | EventSourcedEntity | `application/InboundLeadQueue.java` |
| `LeadsView` | View | `application/LeadsView.java` |
| `InboundLeadConsumer` | Consumer | `application/InboundLeadConsumer.java` |
| `ScoreEvalConsumer` | Consumer | `application/ScoreEvalConsumer.java` |
| `LeadSimulator` | TimedAction | `application/LeadSimulator.java` |
| `LeadEndpoint` | HttpEndpoint | `api/LeadEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |
| `Bootstrap` | service-setup | `Bootstrap.java` |
| `Lead`, `LeadStatus`, `LeadEvent` | domain | `domain/*.java` |

## Concurrency notes

- **Step timeouts (Lesson 4).** `intakeStep`, `researchStep`, `scoreStep`, `strategyStep` set `stepTimeout(60s)` because they call agents. The pipeline is linear with no waiting step.
- **Idempotency.** Each workflow instance is keyed by `leadId`; agent calls use `forAutonomousAgent(Agent.class, role + leadId)` so retries reuse the same session. `recordEvaluation` from `ScoreEvalConsumer` is keyed by `leadId` and is a no-op if the eval is already recorded.
- **Eval decoupling.** `ScoreEvalConsumer` reacts to `LeadScored` asynchronously and does not block the workflow; the strategy step runs independently. The eval result and the strategy may land in either order over SSE.
- **No saga.** All side effects are in-process EventSourcedEntity commands; there is nothing external to compensate. `COMPLETE` is terminal.
- **View indexing (Lesson 2).** `LeadsView` has one query with no `WHERE status` clause; status filtering happens client-side in `LeadEndpoint`.
