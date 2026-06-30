# PLAN — screenplay-writer-marketplace

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
  classDef san fill:#0e1a2a,stroke:#38bdf8,color:#38bdf8;

  API[ScreenplayEndpoint]:::ep
  Entity[ScreenplayEntity]:::ese
  WF[ScreenplayPipelineWorkflow]:::wf
  Agent[ScreenplayAgent]:::agent
  Parse[ParseTools]:::tool
  Develop[DevelopTools]:::tool
  Format[FormatTools]:::tool
  Sanitizer[PiiSanitizer]:::san
  Guard[DeliveryGuardrail]:::guard
  View[ScreenplayView]:::view
  App[AppEndpoint]:::ep

  API -->|sanitize| Sanitizer
  Sanitizer -->|SanitizedSource| API
  API -->|create| Entity
  API -->|start| WF
  WF -->|parseStep runSingleTask| Agent
  Agent -->|invokes| Parse
  Agent -->|invokes| Develop
  Agent -->|invokes| Format
  Agent -.->|before-agent-response| Guard
  Guard -->|blockDelivery| Entity
  Agent -->|ParsedSource / ScenePlan / Screenplay| WF
  WF -->|recordParsedSource / ScenePlan / Screenplay| Entity
  WF -->|deliverStep confirm| Entity
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
  participant API as ScreenplayEndpoint
  participant San as PiiSanitizer
  participant E as ScreenplayEntity
  participant W as ScreenplayPipelineWorkflow
  participant A as ScreenplayAgent
  participant T as Tools (Parse/Develop/Format)
  participant G as DeliveryGuardrail

  U->>API: POST /api/screenplays { sourceTitle, sourceText }
  API->>San: sanitize(sourceText)
  San-->>API: SanitizedSource + PlaceholderMap
  API->>E: create(sourceTitle, sanitizedSource)
  E-->>API: { screenplayId }
  API->>W: start(screenplayId, sourceTitle)
  W->>E: startParse
  W->>A: runSingleTask(PARSE_SOURCE, sanitizedText)
  A->>T: extractCharacters + extractSettings + extractBeats
  T-->>A: List<Character> / List<Setting> / List<Beat>
  A-->>W: ParsedSource
  W->>E: recordParsedSource
  W->>A: runSingleTask(DEVELOP_SCENES, parsedSource)
  A->>T: buildSceneOutline + assignBeatsToScenes
  T-->>A: List<SceneOutline>
  A-->>W: ScenePlan
  W->>E: recordScenePlan
  W->>A: runSingleTask(FORMAT_SCREENPLAY, scenePlan)
  A->>T: formatSlugline + formatAction + formatDialogue
  T-->>A: SceneBlock per scene
  A->>G: before-agent-response(screenplay text)
  G-->>A: accept (no PII detected)
  A-->>W: Screenplay
  W->>E: recordScreenplay
  W->>E: recordDelivery
  E-.->>U: SSE event(DELIVERED)
```

## State machine — `ScreenplayEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> PARSING: ParseStarted
  PARSING --> PARSED: SourceParsed
  PARSED --> DEVELOPING: DevelopStarted
  DEVELOPING --> DEVELOPED: ScenesDevoped
  DEVELOPED --> FORMATTING: FormatStarted
  FORMATTING --> FORMATTED: ScreenplayFormatted
  FORMATTED --> DELIVERED: ScreenplayDelivered
  FORMATTED --> DELIVERY_BLOCKED: DeliveryBlocked
  PARSING --> FAILED: ScreenplayFailed
  DEVELOPING --> FAILED: ScreenplayFailed
  FORMATTING --> FAILED: ScreenplayFailed
  DELIVERED --> [*]
  DELIVERY_BLOCKED --> [*]
  FAILED --> [*]
```

DeliveryBlocked is a terminal state — the author must re-examine the source material and resubmit. DELIVERY_BLOCKED screenplays are never delivered; the guardrail rejection is preserved on the entity for audit.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  ScreenplayEntity ||--o{ ScreenplayCreated : emits
  ScreenplayEntity ||--o{ ParseStarted : emits
  ScreenplayEntity ||--o{ SourceParsed : emits
  ScreenplayEntity ||--o{ DevelopStarted : emits
  ScreenplayEntity ||--o{ ScenesDevoped : emits
  ScreenplayEntity ||--o{ FormatStarted : emits
  ScreenplayEntity ||--o{ ScreenplayFormatted : emits
  ScreenplayEntity ||--o{ ScreenplayDelivered : emits
  ScreenplayEntity ||--o{ DeliveryBlocked : emits
  ScreenplayEntity ||--o{ ScreenplayFailed : emits
  ScreenplayView }o--|| ScreenplayEntity : projects
  ScreenplayPipelineWorkflow }o--|| ScreenplayEntity : reads-and-writes
  ScreenplayAgent ||--o{ ParsedSource : returns
  ScreenplayAgent ||--o{ ScenePlan : returns
  ScreenplayAgent ||--o{ Screenplay : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ScreenplayEndpoint` | `api/ScreenplayEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ScreenplayEntity` | `application/ScreenplayEntity.java` (state in `domain/ScreenplayRecord.java`, events in `domain/ScreenplayEvent.java`) |
| `ScreenplayPipelineWorkflow` | `application/ScreenplayPipelineWorkflow.java` |
| `ScreenplayAgent` | `application/ScreenplayAgent.java` (tasks in `application/ScreenplayTasks.java`) |
| `ParseTools` | `application/ParseTools.java` |
| `DevelopTools` | `application/DevelopTools.java` |
| `FormatTools` | `application/FormatTools.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `DeliveryGuardrail` | `application/DeliveryGuardrail.java` |
| `ScreenplayView` | `application/ScreenplayView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `parseStep` 60 s, `developStep` 60 s, `formatStep` 60 s, `deliverStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ScreenplayPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + screenplayId` as the workflow id; restart of the same screenplayId is rejected by the workflow runtime. The agent instance id is `"agent-" + screenplayId`.
- **One agent per screenplay**: `ScreenplayAgent` runs three tasks per screenplay — PARSE, DEVELOP, FORMAT — each with `capability(...).maxIterationsPerTask(4)`.
- **Guardrail is terminal at the egress boundary**: when `DeliveryGuardrail` rejects the agent's response, the workflow does not retry the FORMAT task — it routes to the error step and records `DeliveryBlocked`. The author must resubmit with cleaner source material.
- **Sanitizer is synchronous at ingestion**: `PiiSanitizer` runs in-thread inside `ScreenplayEndpoint.submitScreenplay()` before any async entity write. Its output is the only form of the source text that ever reaches the agent.
- **Task-boundary handoff is the dependency contract**: `parseStep` writes `SourceParsed` BEFORE returning; `developStep` reads the recorded `ParsedSource` from the entity to build its task's instruction context; `formatStep` reads both `ParsedSource` and `ScenePlan`. The agent is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed screenplay stays at the last successful event; the UI shows the partial state for the user.
