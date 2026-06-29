# Architecture — podcast-summarizer

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `EpisodeEndpoint` accepts a `{episodeTitle, transcriptText}` POST, writes `EpisodeCreated` onto `EpisodeEntity`, and starts `SummaryPipelineWorkflow` keyed by `"pipeline-" + episodeId`. The workflow's first step (`extractStep`) emits `ExtractStarted`, then calls `TranscriptAgent` with `TaskDef.taskType(EXTRACT_QUOTES)` and a `phase = EXTRACT` metadata tag. The agent invokes `ExtractTools.scanTranscript` and `ExtractTools.fetchSpeakerTurn`; every call passes through `PhaseGuardrail` first. Once the agent returns a `QuoteSet`, the workflow writes `QuotesExtracted` onto the entity and advances to `segmentStep` — same pattern, the SEGMENT task carries `phase = SEGMENT`. Then `summarizeStep` runs with `phase = SUMMARIZE`. After `SummaryWritten` lands, `evalStep` runs `FidelityScorer` over the recorded `(QuoteSet, Segmentation, EpisodeSummary)` triple — no LLM call — and writes `EvaluationScored`. `EpisodeView` projects every event into a read-model row; `EpisodeEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `FidelityScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth noting:

1. The task boundary IS the dependency contract. Between `extractStep` and `segmentStep`, the workflow writes `QuotesExtracted` onto the entity. The next step then reads `quotes` from the entity to build the SEGMENT task's instruction context. The agent never sees extract-phase context inside the segment task's conversation; the typed handoff is the only path information travels.
2. Every tool call is filtered through `PhaseGuardrail`. The guardrail reads the in-flight task's `phase` metadata and the current `EpisodeEntity.status`, and applies the per-phase accept matrix. A misordered call is rejected before the tool body executes.

The agent calls themselves are bounded by per-step timeouts (60 s on extract / segment / summarize). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → EXTRACTING → EXTRACTED → SEGMENTING → SEGMENTED → SUMMARIZING → SUMMARIZED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `EXTRACTING`, `SEGMENTING`, or `SUMMARIZING`. A `FAILED` episode's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `PUBLISHED` state. The summary is advisory; the editor consumes it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`EpisodeEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the evaluation, the guardrail audit, the failure, and the initial creation. `EpisodeView` projects every event into a row used by the UI. `SummaryPipelineWorkflow` both reads (`getEpisode`) and writes (`startExtract`, `recordQuotes`, `startSegment`, `recordSegmentation`, `startSummarize`, `recordSummary`, `recordEvaluation`, `recordGuardrailRejection`, `fail`) on the entity. The relationship between `TranscriptAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any episode summary that lands in the entity log, the transcript passed through:

1. **Phase-gate guardrail** — every tool call is filtered. A SUMMARIZE-phase tool called during EXTRACT is rejected before the tool body runs; a `GuardrailRejected` event records the violation for audit.
2. **TranscriptAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **On-decision evaluator** — every emitted summary gets a 1–5 fidelity score. Cluster coverage, quote attribution, quote provenance, and section parity are each worth one point on a base of 1.

Each step is independent. The guardrail does not check fidelity; the evaluator does not check phase order. Removing one of them opens an explicit gap the other does not silently cover.
