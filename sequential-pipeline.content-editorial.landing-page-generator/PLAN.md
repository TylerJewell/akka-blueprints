# PLAN — Landing Page Generator

Architectural sketch for the `sequential-pipeline.content-editorial.landing-page-generator` blueprint. Mermaid sources below are rendered on the Architecture tab of the generated UI. All diagrams use the Akka dark theme plus the Lesson 24 CSS overrides for state-diagram labels (state names forced white; edge-label `foreignObject` set to `overflow:visible`; `transitionLabelColor` `#cccccc`).

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  SIM[ConceptSimulator\nTimedAction] -.->|every 30s| QUEUE[ConceptQueue\nEventSourcedEntity]
  EP[LandingPageEndpoint\nHttpEndpoint] -->|enqueueConcept| QUEUE
  QUEUE -. ConceptQueued .-> CONS[ConceptConsumer\nConsumer]
  CONS -->|start workflow| WF[LandingPageWorkflow\nWorkflow]
  WF -->|runSingleTask| COPY[CopyAgent\nAutonomousAgent]
  WF -->|runSingleTask| STRUCT[StructureAgent\nAutonomousAgent]
  WF -->|runSingleTask| CTA[CtaAgent\nAutonomousAgent]
  WF -->|recordCopy / markReady / markFlagged| ENT[LandingPageEntity\nEventSourcedEntity]
  ENT -. events .-> VIEW[LandingPagesView\nView]
  EP -->|getAllPages| VIEW
  APP[AppEndpoint\nHttpEndpoint] -->|static| UI[index.html]
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions; dotted arrows are scheduled ticks.

## Interaction sequence

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','fontFamily':'Instrument Sans'}}}%%
sequenceDiagram
  participant U as User / Simulator
  participant Q as ConceptQueue
  participant C as ConceptConsumer
  participant W as LandingPageWorkflow
  participant A as Agents
  participant E as LandingPageEntity
  U->>Q: enqueueConcept(concept)
  Q-->>C: ConceptQueued
  C->>W: start(pageId, concept)
  W->>A: CopyAgent.runSingleTask
  A-->>W: CopyDraft
  W->>E: recordCopy
  W->>A: StructureAgent.runSingleTask
  A-->>W: PageSections
  W->>E: recordStructure
  W->>A: CtaAgent.runSingleTask
  A-->>W: CtaBlocks
  W->>E: recordCta
  Note over W: reviewStep — brand-safety guardrail (G1)
  alt passes review
    W->>E: markReady(score)
  else fails review
    W->>E: markFlagged(reason)
  end
```

## State machine

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc','fontFamily':'Instrument Sans'}}}%%
stateDiagram-v2
  [*] --> DRAFTING_COPY
  DRAFTING_COPY --> STRUCTURING: CopyDrafted
  STRUCTURING --> WRITING_CTA: PageStructured
  WRITING_CTA --> REVIEWING: CtaWritten
  REVIEWING --> READY: PageMarkedReady
  REVIEWING --> FLAGGED: PageFlagged
  READY --> [*]
  FLAGGED --> [*]
```

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','fontFamily':'Instrument Sans'}}}%%
erDiagram
  CONCEPT_QUEUE ||--o{ LANDING_PAGE : triggers
  LANDING_PAGE ||--|| PAGES_VIEW : projects
  LANDING_PAGE {
    string id
    string concept
    enum status
    string headline
    string subhead
    string bodyCopy
    list sections
    string ctaPrimary
    string ctaSecondary
    int reviewScore
    string flagReason
  }
  CONCEPT_QUEUE {
    string id
    string concept
  }
  PAGES_VIEW {
    string id
    enum status
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `CopyAgent` | AutonomousAgent | `application/CopyAgent.java` |
| `StructureAgent` | AutonomousAgent | `application/StructureAgent.java` |
| `CtaAgent` | AutonomousAgent | `application/CtaAgent.java` |
| `LandingPageTasks` | task constants | `application/LandingPageTasks.java` |
| `LandingPageWorkflow` | Workflow | `application/LandingPageWorkflow.java` |
| `LandingPageEntity` | EventSourcedEntity | `application/LandingPageEntity.java` |
| `ConceptQueue` | EventSourcedEntity | `application/ConceptQueue.java` |
| `LandingPagesView` | View | `application/LandingPagesView.java` |
| `ConceptConsumer` | Consumer | `application/ConceptConsumer.java` |
| `ConceptSimulator` | TimedAction | `application/ConceptSimulator.java` |
| `LandingPageEndpoint` | HttpEndpoint | `api/LandingPageEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |
| `LandingPage`, records | domain records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** Every workflow step calls an agent (or evaluates the page), so each gets an explicit `stepTimeout(60s)` override — the default 5s timeout would expire mid-LLM-call (Lesson 4).
- **Idempotency.** The workflow is keyed by `pageId` (a fresh UUID minted by `ConceptConsumer`). Re-delivery of the same `ConceptQueued` event re-uses the same page id, so entity commands are naturally idempotent per page.
- **Recovery / compensation.** `defaultStepRecovery(maxRetries(2).failoverTo(error))` retries transient agent failures twice, then routes to a terminal error step. No saga is needed — the pipeline is linear and the only externally visible state is the entity, written once per stage.
- **Review gate.** `reviewStep` is the single compensation point: a failing brand-safety evaluation routes to `markFlagged`, which is terminal and blocks the `READY` transition. This is the G1 guardrail boundary.
