# PLAN — prompt-chaining-workflow

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

  API[DraftEndpoint]:::ep
  Entity[DraftEntity]:::ese
  WF[DraftingWorkflow]:::wf
  Agent[DraftingAgent]:::agent
  Outline[OutlineTools]:::tool
  Draft[DraftTools]:::tool
  Refine[RefineTools]:::tool
  Guard[OutputGuardrail]:::guard
  Scorer[QualityScorer]:::guard
  View[DraftView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|outlineStep runSingleTask| Agent
  Agent -->|invokes| Outline
  Agent -->|invokes| Draft
  Agent -->|invokes| Refine
  Agent -.->|before-agent-response| Guard
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|Outline / Draft / RefinedDocument| WF
  WF -->|recordOutline/Draft/Refinement| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|QualityResult| WF
  WF -->|recordQuality| Entity
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
  participant API as DraftEndpoint
  participant E as DraftEntity
  participant W as DraftingWorkflow
  participant A as DraftingAgent
  participant G as OutputGuardrail
  participant T as Tools (Outline/Draft/Refine)
  participant Sc as QualityScorer

  U->>API: POST /api/drafts { prompt }
  API->>E: create(prompt)
  E-->>API: { draftId }
  API->>W: start(draftId, prompt)
  W->>E: startOutline
  W->>A: runSingleTask(OUTLINE_DOCUMENT, prompt)
  A->>T: structurePrompt + fetchSectionTemplates
  T-->>A: List<Section>
  A-->>W: Outline
  W->>E: recordOutline
  W->>A: runSingleTask(DRAFT_DOCUMENT, outline)
  A->>T: writeSection + gatherCitations
  T-->>A: SectionDraft / List<Citation>
  A-->>W: Draft
  W->>E: recordDraft
  W->>A: runSingleTask(REFINE_DOCUMENT, draft)
  A->>T: polishSection + formatCitation
  T-->>A: RefinedDocument
  A->>G: before-agent-response(RefinedDocument)
  G-->>A: accept (sections>=2, words>=150, citations>=1)
  A-->>W: RefinedDocument
  W->>E: recordRefinement
  W->>Sc: score(refinedDocument, draft, outline)
  Sc-->>W: QualityResult
  W->>E: recordQuality
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `DraftEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> OUTLINING: OutlineStarted
  OUTLINING --> OUTLINED: OutlineProduced
  OUTLINED --> DRAFTING: DraftStarted
  DRAFTING --> DRAFTED: DraftWritten
  DRAFTED --> REFINING: RefineStarted
  REFINING --> REFINED: RefinementApplied
  REFINED --> EVALUATED: QualityScored
  OUTLINING --> FAILED: DraftFailed
  DRAFTING --> FAILED: DraftFailed
  REFINING --> FAILED: DraftFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

`GuardrailRejected` is a side-event recorded on the entity for audit; it does not change the status — the workflow retries `refineStep` and the entity stays in `REFINING`. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  DraftEntity ||--o{ DraftCreated : emits
  DraftEntity ||--o{ OutlineStarted : emits
  DraftEntity ||--o{ OutlineProduced : emits
  DraftEntity ||--o{ DraftStarted : emits
  DraftEntity ||--o{ DraftWritten : emits
  DraftEntity ||--o{ RefineStarted : emits
  DraftEntity ||--o{ RefinementApplied : emits
  DraftEntity ||--o{ QualityScored : emits
  DraftEntity ||--o{ GuardrailRejected : emits
  DraftEntity ||--o{ DraftFailed : emits
  DraftView }o--|| DraftEntity : projects
  DraftingWorkflow }o--|| DraftEntity : reads-and-writes
  DraftingAgent ||--o{ Outline : returns
  DraftingAgent ||--o{ Draft : returns
  DraftingAgent ||--o{ RefinedDocument : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `DraftEndpoint` | `api/DraftEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `DraftEntity` | `application/DraftEntity.java` (state in `domain/DraftRecord.java`, events in `domain/DraftEvent.java`) |
| `DraftingWorkflow` | `application/DraftingWorkflow.java` |
| `DraftingAgent` | `application/DraftingAgent.java` (tasks in `application/DraftingTasks.java`) |
| `OutlineTools` | `application/OutlineTools.java` |
| `DraftTools` | `application/DraftTools.java` |
| `RefineTools` | `application/RefineTools.java` |
| `OutputGuardrail` | `application/OutputGuardrail.java` |
| `QualityScorer` | `application/QualityScorer.java` |
| `DraftView` | `application/DraftView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `outlineStep` 60 s, `draftStep` 90 s, `refineStep` 90 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(DraftingWorkflow::error)`. The 90 s on draft and refine steps accommodates LLM latency for longer document generation (Lesson 4).
- **Idempotency**: each workflow uses `"drafting-" + draftId` as the workflow id; restart of the same draftId is rejected by the workflow runtime. The agent instance id is `"agent-" + draftId` so each draft has its own per-task conversation memory.
- **One agent per draft**: `DraftingAgent` runs three tasks per draft — OUTLINE, DRAFT, REFINE — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget accommodates the guardrail retry path where refineStep must be retried.
- **Guardrail-driven retry on refineStep**: when `OutputGuardrail` rejects a `RefinedDocument`, the rejection reason is appended to the next retry's instruction context and `refineStep` re-invokes `runSingleTask`. The retry counter is tracked in the workflow step; if both retries fail, the workflow steps over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `QualityScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same document always scores the same.
- **Task-boundary handoff is the dependency contract**: `outlineStep` writes `OutlineProduced` BEFORE returning; `draftStep` reads the recorded `Outline` from the entity to build its task's instruction context; `refineStep` reads both `Outline` and `Draft`. The agent itself is stateless across steps.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed draft stays at the last successful event; the UI shows the partial state.
