# PLAN — long-rag-workflow

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

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  WF[LongRagPipelineWorkflow]:::wf
  Agent[DocumentAgent]:::agent
  Retrieve[RetrieveTools]:::tool
  Synthesize[SynthesizeTools]:::tool
  Report[ReportTools]:::tool
  Guard[CitationGuardrail]:::guard
  Scorer[CoverageScorer]:::guard
  View[QueryView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|retrieveStep runSingleTask| Agent
  Agent -.->|after-llm-response| Guard
  Agent -->|invokes| Retrieve
  Agent -->|invokes| Synthesize
  Agent -->|invokes| Report
  Guard -->|recordCitationFlag| Entity
  Agent -->|ChunkWindow / Synthesis / ResearchReport| WF
  WF -->|recordChunks/Synthesis/Report| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|EvalResult| WF
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
  participant API as QueryEndpoint
  participant E as QueryEntity
  participant W as LongRagPipelineWorkflow
  participant A as DocumentAgent
  participant G as CitationGuardrail
  participant T as Tools (Retrieve/Synthesize/Report)
  participant Sc as CoverageScorer

  U->>API: POST /api/queries { queryText }
  API->>E: create(queryText)
  E-->>API: { queryId }
  API->>W: start(queryId, queryText)
  W->>E: startRetrieve
  W->>A: runSingleTask(RETRIEVE_CHUNKS, queryText)
  A->>T: searchChunks + fetchChunkText
  T-->>A: List<Chunk>
  A->>G: after-llm-response(response, chunkWindow)
  G-->>A: accept (all sentences cited)
  A-->>W: ChunkWindow
  W->>E: recordChunks
  W->>A: runSingleTask(SYNTHESIZE_FINDINGS, chunkWindow)
  A->>T: extractFindings + groupFindings
  T-->>A: List<Finding> / List<Theme>
  A->>G: after-llm-response(response, chunkWindow)
  G-->>A: accept
  A-->>W: Synthesis
  W->>E: recordSynthesis
  W->>A: runSingleTask(WRITE_REPORT, synthesis)
  A->>T: draftSection + gatherCitations
  T-->>A: ReportSection / List<Citation>
  A->>G: after-llm-response(response, chunkWindow)
  G-->>A: accept
  A-->>W: ResearchReport
  W->>E: recordReport
  W->>Sc: score(report, synthesis, chunkWindow)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `QueryEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> RETRIEVING: RetrieveStarted
  RETRIEVING --> RETRIEVED: ChunksRetrieved
  RETRIEVED --> SYNTHESIZING: SynthesizeStarted
  SYNTHESIZING --> SYNTHESIZED: FindingsSynthesized
  SYNTHESIZED --> REPORTING: ReportStarted
  REPORTING --> REPORTED: ReportWritten
  REPORTED --> EVALUATED: EvaluationScored
  RETRIEVING --> FAILED: QueryFailed
  SYNTHESIZING --> FAILED: QueryFailed
  REPORTING --> FAILED: QueryFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

CitationFlagged is a side-event recorded on the entity for audit; it does not change the status — the agent's revision stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  QueryEntity ||--o{ QueryCreated : emits
  QueryEntity ||--o{ RetrieveStarted : emits
  QueryEntity ||--o{ ChunksRetrieved : emits
  QueryEntity ||--o{ SynthesizeStarted : emits
  QueryEntity ||--o{ FindingsSynthesized : emits
  QueryEntity ||--o{ ReportStarted : emits
  QueryEntity ||--o{ ReportWritten : emits
  QueryEntity ||--o{ EvaluationScored : emits
  QueryEntity ||--o{ CitationFlagged : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryView }o--|| QueryEntity : projects
  LongRagPipelineWorkflow }o--|| QueryEntity : reads-and-writes
  DocumentAgent ||--o{ ChunkWindow : returns
  DocumentAgent ||--o{ Synthesis : returns
  DocumentAgent ||--o{ ResearchReport : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/QueryRecord.java`, events in `domain/QueryEvent.java`) |
| `LongRagPipelineWorkflow` | `application/LongRagPipelineWorkflow.java` |
| `DocumentAgent` | `application/DocumentAgent.java` (tasks in `application/DocumentTasks.java`) |
| `RetrieveTools` | `application/RetrieveTools.java` |
| `SynthesizeTools` | `application/SynthesizeTools.java` |
| `ReportTools` | `application/ReportTools.java` |
| `CitationGuardrail` | `application/CitationGuardrail.java` |
| `CoverageScorer` | `application/CoverageScorer.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `retrieveStep` 90 s, `synthesizeStep` 90 s, `reportStep` 90 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(LongRagPipelineWorkflow::error)`. The 90 s on each agent-calling step accommodates LLM latency on long-context inputs including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"rag-" + queryId` as the workflow id; restart of the same queryId is rejected by the workflow runtime. The agent instance id is `"agent-" + queryId` so each query has its own per-task conversation memory.
- **One agent per query**: `DocumentAgent` runs three tasks per query — RETRIEVE, SYNTHESIZE, REPORT — each with `capability(...).maxIterationsPerTask(5)`. The 5-iteration budget gives the citation guardrail room to flag a missing citation and still let the agent self-correct.
- **Guardrail-driven revision**: when `CitationGuardrail` returns a citation-gap revision instruction, the instruction is fed back to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 5 iterations fail citation checks, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `CoverageScorer` runs in-process inside `evalStep`. No LLM call — the same report always scores the same. This is the single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `retrieveStep` writes `ChunksRetrieved` BEFORE returning; `synthesizeStep` reads the recorded `ChunkWindow` from the entity to build its task's instruction context; `reportStep` reads both `ChunkWindow` and `Synthesis`. The agent is stateless across phases.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. A failed query stays at the last successful event; the UI shows the partial state.
- **Long-document retrieval**: `RetrieveTools.searchChunks` uses a sliding-window strategy with 20% chunk overlap. The window size (default 512 tokens) and overlap percentage are configurable in `application.conf` under `long-rag.chunk-size` and `long-rag.overlap-pct`. The in-process corpus mock ignores these at runtime but the conf keys are wired for deployer override.
