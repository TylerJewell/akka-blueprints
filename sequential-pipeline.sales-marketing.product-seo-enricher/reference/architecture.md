# Architecture — product-seo-enricher

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `EnrichmentEndpoint` accepts a `{productName}` POST, writes `EnrichmentCreated` onto `EnrichmentEntity`, and starts `EnrichmentPipelineWorkflow` keyed by `"pipeline-" + enrichmentId`. The workflow's first step (`fetchStep`) emits `FetchStarted`, then calls `SeoAgent` with `TaskDef.taskType(FETCH_SERP)` and a `phase = FETCH` metadata tag. The agent invokes `FetchTools.fetchSerpPage` and `FetchTools.fetchResultSnippet`; every call passes through `BrowserGuardrail` first. Once the agent returns a `SerpResult`, the workflow writes `SerpFetched` onto the entity and advances to `analyzeStep` — same pattern, the ANALYZE task carries `phase = ANALYZE`. Then `enrichStep` runs with `phase = ENRICH`. After `EnrichmentWritten` lands, `evalStep` runs `QualityScorer` over the recorded `(SerpResult, SerpAnalysis, ProductEnrichment)` triple — no LLM call — and writes `QualityScored`. `EnrichmentView` projects every event into a read-model row; `EnrichmentEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `QualityScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `fetchStep` and `analyzeStep`, the workflow writes `SerpFetched` onto the entity. The next step then reads `serpResult` from the entity to build the ANALYZE task's instruction context. The agent never sees fetch-phase raw SERP content inside the analyze task's conversation; the typed handoff is the only path information travels.
2. Every tool call is filtered through `BrowserGuardrail`. The guardrail reads the in-flight task's `phase` metadata and the current `EnrichmentEntity.status`, and applies the per-phase accept matrix. A misordered call is rejected before the tool body executes.

The agent calls are bounded by per-step timeouts (60 s on fetch / analyze / enrich). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → FETCHING → FETCHED → ANALYZING → ANALYZED → ENRICHING → ENRICHED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `FETCHING`, `ANALYZING`, or `ENRICHING`. A `FAILED` enrichment's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `PUBLISHED` state. The enrichment is advisory; a content editor reviews the ranked keywords and meta description before they reach the product catalog. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`EnrichmentEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the quality evaluation, the guardrail audit, the failure, and the initial creation. `EnrichmentView` projects every event into a row used by the UI. `EnrichmentPipelineWorkflow` both reads (`getEnrichment`) and writes (`startFetch`, `recordSerpResult`, `startAnalyze`, `recordSerpAnalysis`, `startEnrich`, `recordEnrichment`, `recordQuality`, `recordGuardrailRejection`, `fail`) on the entity. The relationship between `SeoAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any enrichment that lands in the entity log, the product name passed through:

1. **Browser-tool phase-gate guardrail** — every tool call is filtered. An ENRICH-phase tool called during FETCH is rejected before the tool body runs; a `GuardrailRejected` event records the violation for audit. This is the key safety property: raw search-result page content never reaches ANALYZE or ENRICH tool code.
2. **SeoAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **On-decision evaluator** — every emitted enrichment gets a 1–5 quality score. Keyword coverage, meta-description length, competitor grounding, and keyword-list non-emptiness are each worth one point on a base of 1.

Each step is independent. The guardrail does not check keyword quality; the evaluator does not check phase order. Removing one of them opens an explicit gap the other does not silently cover.
