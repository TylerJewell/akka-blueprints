# Architecture — competitor-research-pipeline

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `CompetitorEndpoint` accepts a `{name}` POST, writes `CompetitorCreated` onto `CompetitorEntity`, and starts `CompetitorPipelineWorkflow` keyed by `"pipeline-" + competitorId`. The workflow's first step (`searchStep`) emits `SearchStarted`, then calls `ResearchAgent` with `TaskDef.taskType(SEARCH_COMPETITOR)` and a `phase = SEARCH` metadata tag. The agent invokes `SearchTools.searchCompetitor` and `SearchTools.fetchPage`; every call passes through `NotionWriteGuardrail` first. Once the agent returns a `SearchResultSet`, the workflow writes `ResultsFetched` onto the entity and advances to `summarizeStep` — same pattern, the SUMMARIZE task carries `phase = SUMMARIZE`. Then `publishStep` runs with `phase = PUBLISH`. For the PUBLISH phase, `NotionWriteGuardrail` runs two checks in sequence: the standard phase-gate check (is `SummaryProduced` recorded?), then for `writeToNotion` calls, the schema-and-scope check (required fields present, field types correct, database ID in scope). After `ProfilePublished` lands, `evalStep` runs `PublishQualityScorer` over the recorded `(SearchResultSet, ProfileSummary, CompetitorProfile)` triple — no LLM call — and writes `PublishEvaluated`. `CompetitorView` projects every event into a read-model row; `CompetitorEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component and two external services (Exa.ai for SEARCH, Notion for PUBLISH). In mock mode both externals are replaced by in-process fixtures; the governance mechanisms run identically.

## Interaction sequence

The sequence traces the happy path (J1). Three properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `searchStep` and `summarizeStep`, the workflow writes `ResultsFetched` onto the entity. The next step then reads `searchResults` from the entity to build the SUMMARIZE task's instruction context. The agent never sees SEARCH-phase content inside the SUMMARIZE task's conversation; the typed handoff is the only path information travels.
2. `NotionWriteGuardrail` runs twice in the PUBLISH phase — once for `buildNotionPage` (phase check only) and once for `writeToNotion` (phase check + schema check + scope check). The two-step structure in the PUBLISH task gives the agent a chance to fix the payload between `buildNotionPage` and `writeToNotion` if the schema check would have failed.
3. The agent calls are bounded by per-step timeouts (90 s on search / summarize / publish) to accommodate Exa.ai and Notion latency. `evalStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → SEARCHING → SEARCHED → SUMMARIZING → SUMMARIZED → PUBLISHING → PUBLISHED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `SEARCHING`, `SUMMARIZING`, or `PUBLISHING`. A `FAILED` competitor's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `APPROVED` or `DISTRIBUTED` state. The profile is advisory; the analyst reviews it in Notion and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`CompetitorEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the evaluation, the guardrail audit, the failure, and the initial creation. `CompetitorView` projects every event into a row used by the UI. `CompetitorPipelineWorkflow` both reads (`getCompetitor`) and writes (`startSearch`, `recordResults`, `startSummarize`, `recordSummary`, `startPublish`, `recordProfile`, `recordEvaluation`, `recordGuardrailRejection`, `fail`) on the entity. The relationship between `ResearchAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any competitor profile that lands in the entity log, the name passed through:

1. **Phase-gate + Notion schema/scope guardrail** — every tool call is filtered. A PUBLISH-phase tool called during SEARCH is rejected before the tool body runs; a `writeToNotion` call with a missing required field is rejected before the network call; a write targeting the wrong database is rejected before any Notion record is created. `GuardrailRejected` events record every violation for audit.
2. **ResearchAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **On-decision evaluator** — every emitted profile gets a 1–5 completeness score. Field coverage, source attribution, source provenance, and Notion ref presence are each worth one point on a base of 1.

Each step is independent. The guardrail does not check completeness; the evaluator does not check schema conformance. Removing one of them opens an explicit gap the other does not silently cover.
