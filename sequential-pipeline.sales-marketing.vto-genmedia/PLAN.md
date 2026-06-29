# PLAN — vto-genmedia

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

  API[TryOnEndpoint]:::ep
  Entity[TryOnEntity]:::ese
  WF[VtoPipelineWorkflow]:::wf
  Agent[VtoAgent]:::agent
  Prepare[PrepareTools]:::tool
  Generate[GenerateTools]:::tool
  Validate[ValidateTools]:::tool
  Guard[ImageSafetyGuardrail]:::guard
  Scorer[RenderQualityScorer]:::guard
  View[TryOnView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|prepareStep runSingleTask| Agent
  Agent -.->|after-llm-response| Guard
  Agent -->|invokes| Prepare
  Agent -->|invokes| Generate
  Agent -->|invokes| Validate
  Guard -->|recordSafetyRejection| Entity
  Agent -->|AssetBundle / MediaResult / ValidatedMedia| WF
  WF -->|recordAssets/Media/Validated| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|QualityResult| WF
  WF -->|recordEvaluation| Entity
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
  participant API as TryOnEndpoint
  participant E as TryOnEntity
  participant W as VtoPipelineWorkflow
  participant A as VtoAgent
  participant G as ImageSafetyGuardrail
  participant T as Tools (Prepare/Generate/Validate)
  participant Sc as RenderQualityScorer

  U->>API: POST /api/tryons { garmentId, modelPreset, includeVideo }
  API->>E: create(garmentId, modelPreset, includeVideo)
  E-->>API: { tryOnId }
  API->>W: start(tryOnId, garmentId, modelPreset, includeVideo)
  W->>E: startPrepare
  W->>A: runSingleTask(PREPARE_ASSETS, garmentId+preset)
  A->>T: resolveGarment + resolveModelPreset
  T-->>A: GarmentSpec / ModelPreset
  A-->>W: AssetBundle
  W->>E: recordAssets
  W->>A: runSingleTask(GENERATE_MEDIA, assetBundle)
  A->>T: compositeImage + renderVideoClip
  T-->>A: ImageAsset / VideoAsset
  A->>G: after-llm-response(MediaResult)
  G-->>A: accept (content clean)
  A-->>W: MediaResult
  W->>E: recordMedia
  W->>A: runSingleTask(VALIDATE_OUTPUT, mediaResult+assetBundle)
  A->>T: checkAspectRatio + checkColourFidelity
  T-->>A: DimensionReport / ColourReport
  A-->>W: ValidatedMedia
  W->>E: recordValidated
  W->>Sc: score(validatedMedia, assetBundle)
  Sc-->>W: QualityResult
  W->>E: recordEvaluation
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `TryOnEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> PREPARING: PrepareStarted
  PREPARING --> ASSETS_READY: AssetsResolved
  ASSETS_READY --> GENERATING: GenerateStarted
  GENERATING --> MEDIA_GENERATED: MediaGenerated
  MEDIA_GENERATED --> VALIDATING: ValidateStarted
  VALIDATING --> VALIDATED: MediaValidated
  VALIDATED --> EVALUATED: EvaluationScored
  PREPARING --> FAILED: TryOnFailed
  GENERATING --> FAILED: TryOnFailed
  VALIDATING --> FAILED: TryOnFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

`SafetyRejected` is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  TryOnEntity ||--o{ TryOnCreated : emits
  TryOnEntity ||--o{ PrepareStarted : emits
  TryOnEntity ||--o{ AssetsResolved : emits
  TryOnEntity ||--o{ GenerateStarted : emits
  TryOnEntity ||--o{ MediaGenerated : emits
  TryOnEntity ||--o{ ValidateStarted : emits
  TryOnEntity ||--o{ MediaValidated : emits
  TryOnEntity ||--o{ EvaluationScored : emits
  TryOnEntity ||--o{ SafetyRejected : emits
  TryOnEntity ||--o{ TryOnFailed : emits
  TryOnView }o--|| TryOnEntity : projects
  VtoPipelineWorkflow }o--|| TryOnEntity : reads-and-writes
  VtoAgent ||--o{ AssetBundle : returns
  VtoAgent ||--o{ MediaResult : returns
  VtoAgent ||--o{ ValidatedMedia : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `TryOnEndpoint` | `api/TryOnEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `TryOnEntity` | `application/TryOnEntity.java` (state in `domain/TryOnRecord.java`, events in `domain/TryOnEvent.java`) |
| `VtoPipelineWorkflow` | `application/VtoPipelineWorkflow.java` |
| `VtoAgent` | `application/VtoAgent.java` (tasks in `application/VtoTasks.java`) |
| `PrepareTools` | `application/PrepareTools.java` |
| `GenerateTools` | `application/GenerateTools.java` |
| `ValidateTools` | `application/ValidateTools.java` |
| `ImageSafetyGuardrail` | `application/ImageSafetyGuardrail.java` |
| `RenderQualityScorer` | `application/RenderQualityScorer.java` |
| `TryOnView` | `application/TryOnView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `prepareStep` 60 s, `generateStep` 120 s, `validateStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(VtoPipelineWorkflow::error)`. The 120 s on `generateStep` accommodates image and video generation latency (Lesson 4).
- **Idempotency**: each workflow uses `"vto-pipeline-" + tryOnId` as the workflow id; restart of the same tryOnId is rejected by the workflow runtime. The agent instance id is `"vto-agent-" + tryOnId` so each request has its own per-task conversation memory.
- **One agent per request**: `VtoAgent` runs three tasks per try-on — PREPARE, GENERATE, VALIDATE — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the safety guardrail room to reject an unsafe output and still let the agent self-correct.
- **Safety guardrail-driven retry**: when `ImageSafetyGuardrail` rejects a generated output, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail the safety check, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `RenderQualityScorer` runs in-process inside `evalStep`. No LLM call — the same `ValidatedMedia` always scores the same. This is a deliberate single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `prepareStep` writes `AssetsResolved` BEFORE returning; `generateStep` reads the recorded `AssetBundle` from the entity to build its task's instruction context; `validateStep` reads both `AssetBundle` and `MediaResult`. The agent itself is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed try-on request stays at the last successful event; the UI shows the partial state for the user.
