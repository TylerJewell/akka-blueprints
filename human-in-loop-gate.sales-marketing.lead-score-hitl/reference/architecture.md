# Architecture — Lead Score HITL

The diagrams below are the source the generated system renders on the Architecture tab. All four use the Akka theme variables; the state diagram additionally needs the CSS label overrides from Lesson 24 (state names render black-on-black and edge labels clip without them).

## Component graph

Every inbound lead flows from `LeadSimulator` or `LeadEndpoint` into `InboundLeadQueue`. `InboundLeadConsumer` reacts to each `InboundLeadQueued` event and starts one `LeadScoringWorkflow`. The workflow drives the four agents in sequence and writes their results to `LeadEntity`. `LeadsView` projects entity events for the UI and the ranking monitor.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#F5C518','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  SIM[LeadSimulator] --> Q[InboundLeadQueue]
  EP[LeadEndpoint] --> Q
  Q -.-> CON[InboundLeadConsumer]
  CON --> WF[LeadScoringWorkflow]
  WF --> AC[LeadCollectionAgent]
  WF --> AA[LeadAnalysisAgent]
  WF --> AS[LeadScoringAgent]
  WF --> AO[OutreachAgent]
  WF --> LE[LeadEntity]
  EP --> LE
  LE -.-> V[LeadsView]
  MON[LeadRankingMonitor] --> V
  MON --> LE
  EP --> V
```

## Interaction sequence

The primary journey: a lead is collected (after PII sanitization), analyzed, and scored; the workflow pauses; the ranking monitor shortlists the top leads; a human approves; outreach is drafted under a guardrail.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#F5C518','actorTextColor':'#ffffff','noteTextColor':'#111111','fontFamily':'Instrument Sans'}}}%%
sequenceDiagram
  participant U as User/Simulator
  participant W as LeadScoringWorkflow
  participant A as Agents
  participant L as LeadEntity
  participant M as LeadRankingMonitor
  U->>W: lead enqueued, workflow starts
  W->>A: collect (sanitized) / analyze / score
  A-->>W: results
  W->>L: recordCollected / recordAnalysis / recordScore
  Note over W: awaitReviewStep polls every 5s
  M->>L: markShortlisted (top 3)
  U->>L: approve
  W->>A: draft outreach (guardrail-checked)
  A-->>W: OutreachEmail
  W->>L: recordOutreach (CONTACTED)
```

## State machine

`LeadEntity` lifecycle.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#F5C518','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc','fontFamily':'Instrument Sans'}}}%%
stateDiagram-v2
  [*] --> NEW
  NEW --> COLLECTED
  COLLECTED --> ANALYZED
  ANALYZED --> SCORED
  SCORED --> SHORTLISTED
  SHORTLISTED --> APPROVED
  SHORTLISTED --> REJECTED
  APPROVED --> CONTACTED
  REJECTED --> [*]
  CONTACTED --> [*]
```

## Entity model

`InboundLeadQueue` starts a `LeadScoringWorkflow` per inbound lead; the workflow drives `LeadEntity`, whose events project into `LeadsView`.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#F5C518','fontFamily':'Instrument Sans'}}}%%
erDiagram
  INBOUND_LEAD_QUEUE ||--o{ LEAD : starts_workflow
  LEAD ||--o{ LEAD_EVENT : emits
  LEAD ||--|| LEADS_VIEW : projects
```
