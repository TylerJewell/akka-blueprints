# PLAN — podcast-summarizer

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

  API[EpisodeEndpoint]:::ep
  Entity[EpisodeEntity]:::ese
  WF[SummaryPipelineWorkflow]:::wf
  Agent[TranscriptAgent]:::agent
  Extract[ExtractTools]:::tool
  Segment[SegmentTools]:::tool
  Summarize[SummarizeTools]:::tool
  Guard[PhaseGuardrail]:::guard
  Scorer[FidelityScorer]:::guard
  View[EpisodeView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|extractStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Extract
  Agent -->|invokes| Segment
  Agent -->|invokes| Summarize
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|QuoteSet / Segmentation / EpisodeSummary| WF
  WF -->|recordQuotes/Segmentation/Summary| Entity
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
  participant API as EpisodeEndpoint
  participant E as EpisodeEntity
  participant W as SummaryPipelineWorkflow
  participant A as TranscriptAgent
  participant G as PhaseGuardrail
  participant T as Tools (Extract/Segment/Summarize)
  participant Sc as FidelityScorer

  U->>API: POST /api/episodes { episodeTitle, transcriptText }
  API->>E: create(episodeTitle, transcriptText)
  E-->>API: { episodeId }
  API->>W: start(episodeId, episodeTitle, transcriptText)
  W->>E: startExtract
  W->>A: runSingleTask(EXTRACT_QUOTES, transcript)
  A->>G: before-tool-call(scanTranscript, EXTRACT)
  G-->>A: accept (status EXTRACTING)
  A->>T: scanTranscript + fetchSpeakerTurn
  T-->>A: List<Quote>
  A-->>W: QuoteSet
  W->>E: recordQuotes
  W->>A: runSingleTask(SEGMENT_TOPICS, quotes)
  A->>G: before-tool-call(clusterQuotes, SEGMENT)
  G-->>A: accept (status SEGMENTING and quotes present)
  A->>T: clusterQuotes + labelCluster
  T-->>A: List<TopicCluster>
  A-->>W: Segmentation
  W->>E: recordSegmentation
  W->>A: runSingleTask(WRITE_SUMMARY, segmentation)
  A->>G: before-tool-call(draftSection, SUMMARIZE)
  G-->>A: accept (status SUMMARIZING and segmentation present)
  A->>T: draftSection + writeOverallTakeaway
  T-->>A: SummarySection / String
  A-->>W: EpisodeSummary
  W->>E: recordSummary
  W->>Sc: score(summary, segmentation, quotes)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `EpisodeEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> EXTRACTING: ExtractStarted
  EXTRACTING --> EXTRACTED: QuotesExtracted
  EXTRACTED --> SEGMENTING: SegmentStarted
  SEGMENTING --> SEGMENTED: TopicsSegmented
  SEGMENTED --> SUMMARIZING: SummarizeStarted
  SUMMARIZING --> SUMMARIZED: SummaryWritten
  SUMMARIZED --> EVALUATED: EvaluationScored
  EXTRACTING --> FAILED: EpisodeFailed
  SEGMENTING --> FAILED: EpisodeFailed
  SUMMARIZING --> FAILED: EpisodeFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

GuardrailRejected is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  EpisodeEntity ||--o{ EpisodeCreated : emits
  EpisodeEntity ||--o{ ExtractStarted : emits
  EpisodeEntity ||--o{ QuotesExtracted : emits
  EpisodeEntity ||--o{ SegmentStarted : emits
  EpisodeEntity ||--o{ TopicsSegmented : emits
  EpisodeEntity ||--o{ SummarizeStarted : emits
  EpisodeEntity ||--o{ SummaryWritten : emits
  EpisodeEntity ||--o{ EvaluationScored : emits
  EpisodeEntity ||--o{ GuardrailRejected : emits
  EpisodeEntity ||--o{ EpisodeFailed : emits
  EpisodeView }o--|| EpisodeEntity : projects
  SummaryPipelineWorkflow }o--|| EpisodeEntity : reads-and-writes
  TranscriptAgent ||--o{ QuoteSet : returns
  TranscriptAgent ||--o{ Segmentation : returns
  TranscriptAgent ||--o{ EpisodeSummary : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `EpisodeEndpoint` | `api/EpisodeEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `EpisodeEntity` | `application/EpisodeEntity.java` (state in `domain/EpisodeRecord.java`, events in `domain/EpisodeEvent.java`) |
| `SummaryPipelineWorkflow` | `application/SummaryPipelineWorkflow.java` |
| `TranscriptAgent` | `application/TranscriptAgent.java` (tasks in `application/TranscriptTasks.java`) |
| `ExtractTools` | `application/ExtractTools.java` |
| `SegmentTools` | `application/SegmentTools.java` |
| `SummarizeTools` | `application/SummarizeTools.java` |
| `PhaseGuardrail` | `application/PhaseGuardrail.java` |
| `FidelityScorer` | `application/FidelityScorer.java` |
| `EpisodeView` | `application/EpisodeView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `extractStep` 60 s, `segmentStep` 60 s, `summarizeStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(SummaryPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + episodeId` as the workflow id; restart of the same episodeId is rejected by the workflow runtime. The agent instance id is `"agent-" + episodeId` so each episode has its own per-task conversation memory.
- **One agent per episode**: `TranscriptAgent` runs three tasks per episode — EXTRACT, SEGMENT, SUMMARIZE — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a misordered tool call and still let the agent self-correct.
- **Guardrail-driven retry**: when `PhaseGuardrail` rejects a tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `FidelityScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same summary always scores the same. This is a deliberate single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `extractStep` writes `QuotesExtracted` BEFORE returning; `segmentStep` reads the recorded `QuoteSet` from the entity to build its task's instruction context; `summarizeStep` reads both `QuoteSet` and `Segmentation`. The agent itself is stateless across phases — it never holds extract + segment + summarize context in one conversation.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed episode stays at the last successful event; the UI shows the partial state for the user.
