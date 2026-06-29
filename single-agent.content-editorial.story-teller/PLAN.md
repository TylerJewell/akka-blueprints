# PLAN — story-teller

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[StoryEndpoint]:::ep
  Entity[StoryEntity]:::ese
  Enricher[PromptEnricher]:::cons
  WF[StoryWorkflow]:::wf
  Agent[StoryTellerAgent]:::agent
  Guard[StoryGuardrail]:::guard
  Scorer[QualityScorer]:::guard
  View[StoryView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|StoryRequested| Enricher
  Enricher -->|attachEnriched or block| Entity
  Enricher -->|start workflow| WF
  WF -->|awaitEnrichedStep poll| Entity
  WF -->|generationStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|GeneratedStory| WF
  WF -->|recordStory| Entity
  WF -->|qualityStep score| Scorer
  Scorer -->|QualityResult| WF
  WF -->|recordQuality| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as StoryEndpoint
  participant E as StoryEntity
  participant En as PromptEnricher
  participant W as StoryWorkflow
  participant A as StoryTellerAgent
  participant G as StoryGuardrail
  participant Sc as QualityScorer

  U->>API: POST /api/stories
  API->>E: submit(request)
  E-->>API: { storyId }
  E-.->>En: StoryRequested
  En->>En: content-safety check + tag
  En->>E: attachEnriched
  En->>W: start(storyId)
  W->>E: poll getStory
  E-->>W: enriched.isPresent()
  W->>E: markGenerating
  W->>A: runSingleTask(constraints + attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: GeneratedStory
  W->>E: recordStory(story)
  W->>Sc: score(story, request)
  Sc-->>W: QualityResult
  W->>E: recordQuality(quality)
  E-.->>U: SSE event(SCORED)
```

## State machine — `StoryEntity`

```mermaid
stateDiagram-v2
  [*] --> REQUESTED
  REQUESTED --> ENRICHED: PromptEnriched (safety passed)
  REQUESTED --> BLOCKED: StoryBlocked (safety failed)
  ENRICHED --> GENERATING: GenerationStarted
  GENERATING --> STORY_RECORDED: StoryRecorded
  STORY_RECORDED --> SCORED: QualityScored
  GENERATING --> FAILED: StoryFailed (agent error)
  REQUESTED --> FAILED: StoryFailed (enricher error)
  SCORED --> [*]
  BLOCKED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  StoryEntity ||--o{ StoryRequested : emits
  StoryEntity ||--o{ PromptEnriched : emits
  StoryEntity ||--o{ StoryBlocked : emits
  StoryEntity ||--o{ GenerationStarted : emits
  StoryEntity ||--o{ StoryRecorded : emits
  StoryEntity ||--o{ QualityScored : emits
  StoryEntity ||--o{ StoryFailed : emits
  StoryView }o--|| StoryEntity : projects
  PromptEnricher }o--|| StoryEntity : subscribes
  StoryWorkflow }o--|| StoryEntity : reads-and-writes
  StoryTellerAgent ||--o{ GeneratedStory : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `StoryEndpoint` | `api/StoryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `StoryEntity` | `application/StoryEntity.java` (state in `domain/Story.java`, events in `domain/StoryEvent.java`) |
| `PromptEnricher` | `application/PromptEnricher.java` |
| `StoryWorkflow` | `application/StoryWorkflow.java` |
| `StoryTellerAgent` | `application/StoryTellerAgent.java` (tasks in `application/StoryTasks.java`) |
| `StoryGuardrail` | `application/StoryGuardrail.java` |
| `QualityScorer` | `application/QualityScorer.java` |
| `StoryView` | `application/StoryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitEnrichedStep` 15 s, `generationStep` 60 s, `qualityStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(StoryWorkflow::error)`. The 60 s on `generationStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"story-" + storyId` as the workflow id; the `PromptEnricher` Consumer is allowed to redeliver `StoryRequested` events because `StoryEntity.attachEnriched` is event-version-guarded — a second enrichment attempt against an already-enriched story is a no-op.
- **One agent per story**: the AutonomousAgent instance id is `"teller-" + storyId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `StoryGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `generationStep` fails over to `error` and the entity transitions to `FAILED`.
- **Quality eval is synchronous and deterministic**: `QualityScorer` runs in-process inside `qualityStep`. No LLM call, no external service — the same story always scores the same. This is a deliberate single-agent guarantee.
- **Blocked path skips agent entirely**: when `PromptEnricher` calls `StoryEntity.block(reason)`, the workflow's `awaitEnrichedStep` detects `status == BLOCKED` and transitions to terminal done without invoking `StoryTellerAgent`. No model credit consumed.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
