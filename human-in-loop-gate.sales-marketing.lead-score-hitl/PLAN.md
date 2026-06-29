# Implementation Plan — Lead Score HITL

The architecture `SPEC.md` resolves to once run through `/akka:specify` → `/akka:plan`. Diagrams render on the Architecture tab; they carry the Akka theme variables plus the CSS overrides for state-diagram labels and edge-label `foreignObject` overflow (Lesson 24).

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#F5C518','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  SIM[LeadSimulator\nTimedAction]
  Q[InboundLeadQueue\nEventSourcedEntity]
  CON[InboundLeadConsumer\nConsumer]
  WF[LeadScoringWorkflow\nWorkflow]
  AC[LeadCollectionAgent\nAutonomousAgent]
  AA[LeadAnalysisAgent\nAutonomousAgent]
  AS[LeadScoringAgent\nAutonomousAgent]
  AO[OutreachAgent\nAutonomousAgent]
  LE[LeadEntity\nEventSourcedEntity]
  V[LeadsView\nView]
  MON[LeadRankingMonitor\nTimedAction]
  EP[LeadEndpoint\nHttpEndpoint]
  APP[AppEndpoint\nHttpEndpoint]

  SIM -->|enqueueLead| Q
  EP -->|enqueueLead| Q
  Q -.InboundLeadQueued.-> CON
  CON -->|start| WF
  WF -->|runSingleTask| AC
  WF -->|runSingleTask| AA
  WF -->|runSingleTask| AS
  WF -->|runSingleTask| AO
  WF -->|commands| LE
  EP -->|approve/reject| LE
  LE -.events.-> V
  MON -->|getAllLeads| V
  MON -->|markShortlisted| LE
  EP -->|queries/SSE| V
  APP -->|static| APP
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions.

## Interaction sequence

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#F5C518','actorTextColor':'#ffffff','noteTextColor':'#111111','fontFamily':'Instrument Sans'}}}%%
sequenceDiagram
  participant U as User/Simulator
  participant Q as InboundLeadQueue
  participant W as LeadScoringWorkflow
  participant A as Agents
  participant L as LeadEntity
  participant M as LeadRankingMonitor
  U->>Q: enqueueLead(rawSource)
  Q-->>W: start(leadId)
  W->>A: collect (PII sanitized first)
  A-->>W: CollectedLead
  W->>L: recordCollected
  W->>A: analyze
  A-->>W: LeadAnalysis
  W->>L: recordAnalysis
  W->>A: score
  A-->>W: LeadScore
  W->>L: recordScore (SCORED)
  Note over W: awaitReviewStep polls every 5s
  M->>L: markShortlisted (top 3)
  U->>L: approve
  W->>A: outreach (guardrail-checked)
  A-->>W: OutreachEmail
  W->>L: recordOutreach (CONTACTED)
```

## State machine

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#F5C518','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc','fontFamily':'Instrument Sans'}}}%%
stateDiagram-v2
  [*] --> NEW
  NEW --> COLLECTED: recordCollected
  COLLECTED --> ANALYZED: recordAnalysis
  ANALYZED --> SCORED: recordScore
  SCORED --> SHORTLISTED: markShortlisted
  SHORTLISTED --> APPROVED: approve
  SHORTLISTED --> REJECTED: reject
  APPROVED --> CONTACTED: recordOutreach
  REJECTED --> [*]
  CONTACTED --> [*]
```

State labels need the CSS overrides from Lesson 24 — the `g.statediagram-state .label` path does not inherit `primaryTextColor`, and edge labels clip without `overflow:visible` on their `foreignObject`.

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
| `LeadCollectionAgent` | AutonomousAgent | `application/LeadCollectionAgent.java` |
| `LeadAnalysisAgent` | AutonomousAgent | `application/LeadAnalysisAgent.java` |
| `LeadScoringAgent` | AutonomousAgent | `application/LeadScoringAgent.java` |
| `OutreachAgent` | AutonomousAgent | `application/OutreachAgent.java` |
| `LeadScoringTasks` | task definitions | `application/LeadScoringTasks.java` |
| `PiiSanitizer` | helper | `application/PiiSanitizer.java` |
| `LeadScoringWorkflow` | Workflow | `application/LeadScoringWorkflow.java` |
| `LeadEntity` | EventSourcedEntity | `application/LeadEntity.java` |
| `InboundLeadQueue` | EventSourcedEntity | `application/InboundLeadQueue.java` |
| `LeadsView` | View | `application/LeadsView.java` |
| `InboundLeadConsumer` | Consumer | `application/InboundLeadConsumer.java` |
| `LeadSimulator` | TimedAction | `application/LeadSimulator.java` |
| `LeadRankingMonitor` | TimedAction | `application/LeadRankingMonitor.java` |
| `LeadEndpoint` | HttpEndpoint | `api/LeadEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |
| `Bootstrap` | service-setup | `Bootstrap.java` |
| `Lead`, `LeadStatus`, `LeadEvent` | domain | `domain/*.java` |

## Concurrency notes

- **Step timeouts (Lesson 4).** `collectStep`, `analyzeStep`, `scoreStep`, `outreachStep` set `stepTimeout(60s)` because they call agents. `awaitReviewStep` uses `stepTimeout(10s)` and self-schedules a 5s resume timer while the lead is `SCORED` or `SHORTLISTED`.
- **Idempotency.** Each workflow instance is keyed by `leadId`; agent calls use `forAutonomousAgent(Agent.class, role + leadId)` so retries reuse the same session. `markShortlisted` is a no-op if the lead is already past `SCORED`.
- **No saga.** All side effects are in-process EventSourcedEntity commands; there is nothing external to compensate. A rejected lead is terminal; an approved lead drafts outreach exactly once because `recordOutreach` is only emitted from `outreachStep`.
- **View indexing (Lesson 2).** `LeadsView` has one query with no `WHERE status` clause; status and top-3 ranking are filtered and sorted client-side in `LeadEndpoint` and `LeadRankingMonitor`.
