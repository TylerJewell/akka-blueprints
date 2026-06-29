# PLAN — Meeting Prep Briefer

Architecture sketch for the delegation-supervisor-workers x research-intel cell. `/akka:plan` may regenerate this; it is authored here so the generator can verify against it.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1a1a1a','primaryTextColor':'#ffffff','primaryBorderColor':'#e8c547','lineColor':'#e8c547','fontFamily':'Instrument Sans, sans-serif'}}}%%
flowchart TB
  RS[RequestSimulator<br/>TimedAction] -. drips .-> RQ[RequestQueue<br/>EventSourcedEntity]
  EP[BriefingEndpoint<br/>HttpEndpoint] -- enqueue --> RQ
  RQ -. events .-> C[BriefingRequestConsumer<br/>Consumer]
  C -- start --> WF[BriefingWorkflow<br/>Workflow]
  WF -- runSingleTask --> SUP[BriefingSupervisor<br/>AutonomousAgent]
  WF -- runSingleTask --> PR[ParticipantResearcher<br/>AutonomousAgent]
  WF -- runSingleTask --> TR[TopicResearcher<br/>AutonomousAgent]
  WF -- compose --> CO[BriefingComposer<br/>Agent]
  PR -- GET --> SRC[ResearchSource<br/>HttpEndpoint]
  TR -- GET --> SRC
  WF -- commands --> BE[BriefingEntity<br/>EventSourcedEntity]
  BE -. events .-> BV[BriefingsView<br/>View]
  EP -- query/SSE --> BV
  APP[AppEndpoint<br/>HttpEndpoint] --> UI[(static-resources)]
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions; dotted arrows are scheduled ticks.

## Interaction sequence

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1a1a1a','primaryTextColor':'#ffffff','primaryBorderColor':'#e8c547','lineColor':'#e8c547','fontFamily':'Instrument Sans, sans-serif'}}}%%
sequenceDiagram
  participant U as User/Simulator
  participant Q as RequestQueue
  participant W as BriefingWorkflow
  participant S as BriefingSupervisor
  participant R as Workers
  participant E as BriefingEntity
  U->>Q: enqueue(meeting request)
  Q-->>W: consumer starts workflow
  W->>S: PLAN task
  S-->>W: ResearchPlan
  Note over W: researchStep fans out one task per participant + topic
  W->>R: RESEARCH_PARTICIPANT / RESEARCH_TOPIC
  R-->>W: profiles + topic brief
  Note over W: PII sanitizer redacts before persist
  W->>E: addProfile / setTopicBrief
  W->>E: recordComposition (BriefingComposer output)
  E-->>U: SSE Briefing READY
```

## State machine

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1a1a1a','primaryTextColor':'#ffffff','primaryBorderColor':'#e8c547','lineColor':'#e8c547','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc','fontFamily':'Instrument Sans, sans-serif'}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> PLANNING: workflow starts
  PLANNING --> RESEARCHING: ResearchPlanned
  RESEARCHING --> COMPOSING: all profiles + topic brief in
  COMPOSING --> READY: BriefingComposed
  PLANNING --> FAILED: step error
  RESEARCHING --> FAILED: step error
  COMPOSING --> FAILED: step error
  READY --> [*]
  FAILED --> [*]
```

State-label CSS overrides from Lesson 24 must accompany this diagram in the generated `index.html` (state names white-on-dark, edge labels `overflow:visible`).

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1a1a1a','primaryTextColor':'#ffffff','primaryBorderColor':'#e8c547','lineColor':'#e8c547','fontFamily':'Instrument Sans, sans-serif'}}}%%
erDiagram
  BRIEFING ||--o{ PARTICIPANT_PROFILE : contains
  BRIEFING ||--o| TOPIC_BRIEF : contains
  BRIEFING {
    string id
    string meetingTopic
    enum status
    instant readyAt
  }
  PARTICIPANT_PROFILE {
    string name
    string role
    string redactedBackground
  }
  TOPIC_BRIEF {
    string summary
    list keyPoints
  }
  REQUEST_QUEUE {
    string id
    string meetingTopic
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| BriefingSupervisor | AutonomousAgent | `application/BriefingSupervisor.java` |
| ParticipantResearcher | AutonomousAgent | `application/ParticipantResearcher.java` |
| TopicResearcher | AutonomousAgent | `application/TopicResearcher.java` |
| BriefingComposer | Agent | `application/BriefingComposer.java` |
| BriefingTasks | Task constants | `application/BriefingTasks.java` |
| BriefingWorkflow | Workflow | `application/BriefingWorkflow.java` |
| BriefingEntity | EventSourcedEntity | `domain/BriefingEntity.java` |
| RequestQueue | EventSourcedEntity | `domain/RequestQueue.java` |
| BriefingsView | View | `application/BriefingsView.java` |
| BriefingRequestConsumer | Consumer | `application/BriefingRequestConsumer.java` |
| RequestSimulator | TimedAction | `application/RequestSimulator.java` |
| ResearchSource | HttpEndpoint | `api/ResearchSource.java` |
| BriefingEndpoint | HttpEndpoint | `api/BriefingEndpoint.java` |
| AppEndpoint | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `planStep` 60s, `researchStep` 120s (fan-out across multiple agent calls), `composeStep` 60s. `defaultStepRecovery(maxRetries(2).failoverTo(error))`.
- **Idempotency:** the workflow uses the briefing UUID as its instance id; the consumer derives a deterministic id from the queued request offset so a redelivered queue event does not start a duplicate workflow.
- **Fan-out gather:** `researchStep` issues one `RESEARCH_PARTICIPANT` task per planned participant plus one `RESEARCH_TOPIC` task, awaits each `forTask(taskId).result(...)`, sanitizes, then records. A failed worker task fails the step into the recovery path that marks the briefing `FAILED` with a reason — no compensation is needed because nothing external was written.
- **Guardrail placement (G1):** the before-tool-call guardrail sits on the researcher agents and rejects a `ResearchSource` call whose subject is not in the active request, returning a blocked result the agent surfaces as an empty profile rather than retrying forever.
