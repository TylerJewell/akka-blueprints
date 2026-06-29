# PLAN — chain-workflow

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

  API[DocumentEndpoint]:::ep
  Entity[DocumentEntity]:::ese
  WF[TransformPipelineWorkflow]:::wf
  Agent[TransformAgent]:::agent
  Extract[ExtractTools]:::tool
  Refine[RefineTools]:::tool
  Format[FormatTools]:::tool
  Guard[OutputGuardrail]:::guard
  Scorer[QualityScorer]:::guard
  View[DocumentView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|extractStep runSingleTask| Agent
  Agent -.->|after-llm-response| Guard
  Agent -->|invokes| Extract
  Agent -->|invokes| Refine
  Agent -->|invokes| Format
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|ExtractedStructure / RefinedContent / Document| WF
  WF -->|recordExtracted/Refined/Document| Entity
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
  participant API as DocumentEndpoint
  participant E as DocumentEntity
  participant W as TransformPipelineWorkflow
  participant A as TransformAgent
  participant G as OutputGuardrail
  participant T as Tools (Extract/Refine/Format)
  participant Sc as QualityScorer

  U->>API: POST /api/documents { rawInput }
  API->>E: create(rawInput)
  E-->>API: { documentId }
  API->>W: start(documentId, rawInput)
  W->>E: startExtract
  W->>A: runSingleTask(EXTRACT_STRUCTURE, rawInput)
  A->>T: parseInput + classifyFacts
  T-->>A: List<Fact>
  A-->>G: after-llm-response(ExtractedStructure, EXTRACT)
  G-->>W: accept (facts non-empty, factIds valid)
  W->>E: recordExtracted
  W->>A: runSingleTask(REFINE_CONTENT, extracted)
  A->>T: polishFact + mergeDuplicates
  T-->>A: List<RefinedFact>
  A-->>G: after-llm-response(RefinedContent, REFINE)
  G-->>W: accept (sourceFactIds valid)
  W->>E: recordRefined
  W->>A: runSingleTask(FORMAT_DOCUMENT, refined)
  A->>T: buildSection + composeSummary
  T-->>A: List<Section> + summary
  A-->>G: after-llm-response(Document, FORMAT)
  G-->>W: accept (citations non-empty, title present)
  W->>E: recordDocument
  W->>Sc: score(document, refined, extracted)
  Sc-->>W: QualityResult
  W->>E: recordQuality
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `DocumentEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> EXTRACTING: ExtractStarted
  EXTRACTING --> EXTRACTED: StructureExtracted
  EXTRACTED --> REFINING: RefineStarted
  REFINING --> REFINED: ContentRefined
  REFINED --> FORMATTING: FormatStarted
  FORMATTING --> FORMATTED: DocumentFormatted
  FORMATTED --> EVALUATED: QualityScored
  EXTRACTING --> FAILED: DocumentFailed
  REFINING --> FAILED: DocumentFailed
  FORMATTING --> FAILED: DocumentFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

`GuardrailRejected` is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  DocumentEntity ||--o{ DocumentCreated : emits
  DocumentEntity ||--o{ ExtractStarted : emits
  DocumentEntity ||--o{ StructureExtracted : emits
  DocumentEntity ||--o{ RefineStarted : emits
  DocumentEntity ||--o{ ContentRefined : emits
  DocumentEntity ||--o{ FormatStarted : emits
  DocumentEntity ||--o{ DocumentFormatted : emits
  DocumentEntity ||--o{ QualityScored : emits
  DocumentEntity ||--o{ GuardrailRejected : emits
  DocumentEntity ||--o{ DocumentFailed : emits
  DocumentView }o--|| DocumentEntity : projects
  TransformPipelineWorkflow }o--|| DocumentEntity : reads-and-writes
  TransformAgent ||--o{ ExtractedStructure : returns
  TransformAgent ||--o{ RefinedContent : returns
  TransformAgent ||--o{ Document : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `DocumentEndpoint` | `api/DocumentEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `DocumentEntity` | `application/DocumentEntity.java` (state in `domain/DocumentRecord.java`, events in `domain/DocumentEvent.java`) |
| `TransformPipelineWorkflow` | `application/TransformPipelineWorkflow.java` |
| `TransformAgent` | `application/TransformAgent.java` (tasks in `application/TransformTasks.java`) |
| `ExtractTools` | `application/ExtractTools.java` |
| `RefineTools` | `application/RefineTools.java` |
| `FormatTools` | `application/FormatTools.java` |
| `OutputGuardrail` | `application/OutputGuardrail.java` |
| `QualityScorer` | `application/QualityScorer.java` |
| `DocumentView` | `application/DocumentView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `extractStep` 60 s, `refineStep` 60 s, `formatStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(TransformPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"chain-" + documentId` as the workflow id; restart of the same documentId is rejected by the workflow runtime. The agent instance id is `"agent-" + documentId` so each document has its own per-task conversation memory.
- **One agent per document**: `TransformAgent` runs three tasks per document — EXTRACT, REFINE, FORMAT — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject an invalid output and still let the agent self-correct.
- **Guardrail-driven retry**: when `OutputGuardrail` rejects an output, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `QualityScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same document always scores the same. This is a deliberate single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `extractStep` writes `StructureExtracted` BEFORE returning; `refineStep` reads the recorded `ExtractedStructure` from the entity to build its task's instruction context; `formatStep` reads both `ExtractedStructure` and `RefinedContent`. The agent itself is stateless across stages — it never holds extract + refine + format context in one conversation.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed document stays at the last successful event; the UI shows the partial state for the user.
