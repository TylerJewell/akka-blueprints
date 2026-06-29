# Architecture — long-rag-workflow

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence, with long-document citation integrity enforced after every model response. `QueryEndpoint` accepts a `{queryText}` POST, writes `QueryCreated` onto `QueryEntity`, and starts `LongRagPipelineWorkflow` keyed by `"rag-" + queryId`. The workflow's first step (`retrieveStep`) emits `RetrieveStarted`, then calls `DocumentAgent` with `TaskDef.taskType(RETRIEVE_CHUNKS)` and the query text as instruction context. The agent invokes `RetrieveTools.searchChunks` and `RetrieveTools.fetchChunkText`, assembling a `ChunkWindow`. Every model response that the agent generates passes through `CitationGuardrail` before it is returned to the workflow — the guardrail checks that every non-trivial sentence carries a `[chunkId]` citation tag referencing a chunk in the active window.

Once the agent returns a `ChunkWindow`, the workflow writes `ChunksRetrieved` onto the entity and advances to `synthesizeStep`. The SYNTHESIZE task receives the `ChunkWindow` as its instruction context and calls `SynthesizeTools.extractFindings` and `SynthesizeTools.groupFindings` to produce a `Synthesis`. Then `reportStep` runs: the REPORT task receives both `ChunkWindow` and `Synthesis`, calls `ReportTools.draftSection` and `ReportTools.gatherCitations`, and produces the typed `ResearchReport`. After `ReportWritten` lands, `evalStep` runs `CoverageScorer` over the recorded `(ChunkWindow, Synthesis, ResearchReport)` triple — no LLM call — and writes `EvaluationScored`. `QueryView` projects every event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `CoverageScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `retrieveStep` and `synthesizeStep`, the workflow writes `ChunksRetrieved` onto the entity. The next step then reads `chunkWindow` from the entity to build the SYNTHESIZE task's instruction context. The agent never sees retrieve-phase tool results inside the synthesize task's conversation; the typed handoff is the only path information travels.
2. Every model response is filtered through `CitationGuardrail`. The guardrail receives the raw response text and the active `ChunkWindow` chunk-id set from `TaskDef.metadata`, scans for `[chunkId]` citation tags, and either accepts or returns a citation-gap revision instruction. A response with uncited sentences is held before it leaves the agent.

The agent calls themselves are bounded by per-step timeouts (90 s on retrieve / synthesize / report, accommodating long-context generation latency). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → RETRIEVING → RETRIEVED → SYNTHESIZING → SYNTHESIZED → REPORTING → REPORTED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `RETRIEVING`, `SYNTHESIZING`, or `REPORTING`. A `FAILED` query's prior data is preserved on the entity — the UI shows the partial state.
- `CitationFlagged` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `PUBLISHED` or `APPROVED` state. The research report is advisory; the reader consumes it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`QueryEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the evaluation, the citation audit, the failure, and the initial creation. `QueryView` projects every event into a row used by the UI. `LongRagPipelineWorkflow` both reads (`getQuery`) and writes (`startRetrieve`, `recordChunks`, `startSynthesize`, `recordSynthesis`, `startReport`, `recordReport`, `recordEvaluation`, `recordCitationFlag`, `fail`) on the entity. The relationship between `DocumentAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any research report that lands in the entity log, the query passed through:

1. **Citation guardrail** — every model response is filtered before it leaves the agent. A response with uncited sentences is held and the agent is instructed to revise; a `CitationFlagged` event records the revision request for audit.
2. **DocumentAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **On-decision evaluator** — every emitted report gets a 1–5 coverage score. Theme coverage, citation density, chunk provenance, and section parity are each worth one point on a base of 1.

Each step is independent. The citation guardrail does not check coverage density; the evaluator does not check per-sentence grounding. Removing one of them opens an explicit gap the other does not silently cover.
