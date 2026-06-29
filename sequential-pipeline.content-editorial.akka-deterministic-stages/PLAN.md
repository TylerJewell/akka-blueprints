# PLAN — deterministic-multi-stage-agent-pipeline

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef tool fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[StoryEndpoint]:::ep
  Entity[StoryEntity]:::ese
  WF[StoryPipelineWorkflow]:::wf
  Agent[StoryAgent]:::agent
  Outline[OutlineTools]:::tool
  Body[BodyTools]:::tool
  Ending[EndingTools]:::tool
  Guard[StageOutputGuardrail]:::guard
  Validator[StoryStructureValidator]:::guard
  View[StoryView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|outlineStep runSingleTask| Agent
  Agent -.->|after-llm-response| Guard
  Agent -->|invokes| Outline
  Agent -->|invokes| Body
  Agent -->|invokes| Ending
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|Outline / Body / Ending| WF
  WF -->|recordOutline/Body/Ending| Entity
  WF -->|validateStep score| Validator
  Validator -->|ValidationResult| WF
  WF -->|recordValidation| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as StoryEndpoint
  participant E as StoryEntity
  participant W as StoryPipelineWorkflow
  participant A as StoryAgent
  participant G as StageOutputGuardrail
  participant T as Tools (Outline/Body/Ending)
  participant V as StoryStructureValidator

  U->>API: POST /api/stories { prompt }
  API->>E: create(prompt)
  E-->>API: { storyId }
  API->>W: start(storyId, prompt)
  W->>E: startOutline
  W->>A: runSingleTask(OUTLINE_STORY, prompt)
  A->>T: classifyGenre + generateBeats
  T-->>A: genre / List<Beat>
  A-->>G: after-llm-response(Outline)
  G-->>A: accept (beats.size() >= 2)
  A-->>W: Outline
  W->>E: recordOutline
  W->>A: runSingleTask(WRITE_BODY, outline)
  A->>T: expandBeat (per beat) + linkParagraphs
  T-->>A: List<Paragraph>
  A-->>G: after-llm-response(Body)
  G-->>A: accept (all beatIds present in outline)
  A-->>W: Body
  W->>E: recordBody
  W->>A: runSingleTask(WRITE_ENDING, body+outline)
  A->>T: resolveArcs + composeClosure
  T-->>A: List<Arc> / closingText
  A-->>G: after-llm-response(Ending)
  G-->>A: accept (closingText non-empty, arcs present)
  A-->>W: Ending
  W->>E: recordEnding
  W->>V: validate(outline, body, ending)
  V-->>W: ValidationResult
  W->>E: recordValidation
  E-.->>U: SSE event(VALIDATED)
```

## State machine — `StoryEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> OUTLINING: OutlineStarted
  OUTLINING --> OUTLINED: OutlineProduced
  OUTLINED --> BODY_WRITING: BodyStarted
  BODY_WRITING --> BODY_WRITTEN: BodyWritten
  BODY_WRITTEN --> ENDING_WRITING: EndingStarted
  ENDING_WRITING --> ENDING_WRITTEN: EndingWritten
  ENDING_WRITTEN --> VALIDATED: StoryValidated
  OUTLINING --> FAILED: StoryFailed
  BODY_WRITING --> FAILED: StoryFailed
  ENDING_WRITING --> FAILED: StoryFailed
  VALIDATED --> [*]
  FAILED --> [*]
```

`GuardrailRejected` is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  StoryEntity ||--o{ StoryCreated : emits
  StoryEntity ||--o{ OutlineStarted : emits
  StoryEntity ||--o{ OutlineProduced : emits
  StoryEntity ||--o{ BodyStarted : emits
  StoryEntity ||--o{ BodyWritten : emits
  StoryEntity ||--o{ EndingStarted : emits
  StoryEntity ||--o{ EndingWritten : emits
  StoryEntity ||--o{ StoryValidated : emits
  StoryEntity ||--o{ GuardrailRejected : emits
  StoryEntity ||--o{ StoryFailed : emits
  StoryView }o--|| StoryEntity : projects
  StoryPipelineWorkflow }o--|| StoryEntity : reads-and-writes
  StoryAgent ||--o{ Outline : returns
  StoryAgent ||--o{ Body : returns
  StoryAgent ||--o{ Ending : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `StoryEndpoint` | `api/StoryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `StoryEntity` | `application/StoryEntity.java` (state in `domain/StoryRecord.java`, events in `domain/StoryEvent.java`) |
| `StoryPipelineWorkflow` | `application/StoryPipelineWorkflow.java` |
| `StoryAgent` | `application/StoryAgent.java` (tasks in `application/StoryTasks.java`) |
| `OutlineTools` | `application/OutlineTools.java` |
| `BodyTools` | `application/BodyTools.java` |
| `EndingTools` | `application/EndingTools.java` |
| `StageOutputGuardrail` | `application/StageOutputGuardrail.java` |
| `StoryStructureValidator` | `application/StoryStructureValidator.java` |
| `StoryView` | `application/StoryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `outlineStep` 60 s, `bodyStep` 60 s, `endingStep` 60 s, `validateStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(StoryPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + storyId` as the workflow id; restart of the same storyId is rejected by the workflow runtime. The agent instance id is `"agent-" + storyId` so each story has its own per-task conversation memory.
- **One agent per story**: `StoryAgent` runs three tasks per story — OUTLINE, WRITE_BODY, WRITE_ENDING — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a structurally incomplete result and still let the agent self-correct.
- **Guardrail-driven retry**: when `StageOutputGuardrail` rejects a result, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Validation is synchronous and deterministic**: `StoryStructureValidator` runs in-process inside `validateStep`. No LLM call, no external service — the same story always scores the same. This is a deliberate single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `outlineStep` writes `OutlineProduced` BEFORE returning; `bodyStep` reads the recorded `Outline` from the entity to build its task's instruction context; `endingStep` reads both `Outline` and `Body`. The agent itself is stateless across stages — it never holds outline + body + ending context in one conversation.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed story stays at the last successful event; the UI shows the partial state for the user.
