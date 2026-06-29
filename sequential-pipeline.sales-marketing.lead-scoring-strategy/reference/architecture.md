# Architecture — Lead Scoring Strategy

The diagrams below are the source the generated system renders on the Architecture tab. All four use the Akka theme variables; the state diagram additionally needs the CSS label overrides from Lesson 24 (state names render black-on-black and edge labels clip without them).

## Component graph

Every inbound lead flows from `LeadSimulator` or `LeadEndpoint` into `InboundLeadQueue`. `InboundLeadConsumer` reacts to each `InboundLeadQueued` event and starts one `LeadStrategyWorkflow`. The workflow drives the four agents in sequence and writes their results to `LeadEntity`. `ScoreEvalConsumer` reacts to the `LeadScored` event and records an automatic eval. `LeadsView` projects entity events for the UI.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#F5C518','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  SIM[LeadSimulator] --> Q[InboundLeadQueue]
  EP[LeadEndpoint] --> Q
  Q -.-> CON[InboundLeadConsumer]
  CON --> WF[LeadStrategyWorkflow]
  WF --> AI[IntakeAgent]
  WF --> AR[ResearchAgent]
  WF --> AS[ScoringAgent]
  WF --> AT[StrategyAgent]
  WF --> LE[LeadEntity]
  LE -.-> EV[ScoreEvalConsumer]
  EV --> LE
  LE -.-> V[LeadsView]
  EP --> V
```

## Interaction sequence

The primary journey: a lead is taken in (after PII sanitization), researched, and scored; an eval fires on the score; a strategy is drafted; the lead completes.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#F5C518','actorTextColor':'#ffffff','noteTextColor':'#111111','fontFamily':'Instrument Sans'}}}%%
sequenceDiagram
  participant U as User/Simulator
  participant W as LeadStrategyWorkflow
  participant A as Agents
  participant L as LeadEntity
  participant E as ScoreEvalConsumer
  U->>W: lead enqueued, workflow starts
  W->>A: intake (sanitized) / research / score
  A-->>W: results
  W->>L: recordIntake / recordResearch / recordScore
  L-->>E: LeadScored
  E->>L: recordEvaluation (eval result)
  W->>A: draft strategy
  A-->>W: EngagementStrategy
  W->>L: recordStrategy (COMPLETE)
```

## State machine

`LeadEntity` lifecycle. `recordEvaluation` fires on `SCORED` and sets the eval fields without changing the status.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#F5C518','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc','fontFamily':'Instrument Sans'}}}%%
stateDiagram-v2
  [*] --> NEW
  NEW --> INTAKE
  INTAKE --> RESEARCHED
  RESEARCHED --> SCORED
  SCORED --> COMPLETE
  COMPLETE --> [*]
```

## Entity model

`InboundLeadQueue` starts a `LeadStrategyWorkflow` per inbound lead; the workflow drives `LeadEntity`, whose events project into `LeadsView`.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#F5C518','fontFamily':'Instrument Sans'}}}%%
erDiagram
  INBOUND_LEAD_QUEUE ||--o{ LEAD : starts_workflow
  LEAD ||--o{ LEAD_EVENT : emits
  LEAD ||--|| LEADS_VIEW : projects
```
