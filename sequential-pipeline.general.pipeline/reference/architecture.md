# Architecture — pipeline

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `BriefingEndpoint` accepts a `{topic}` POST, writes `BriefingCreated` onto `BriefingEntity`, and starts `BriefingPipelineWorkflow` keyed by `"pipeline-" + briefingId`. The workflow's first step (`collectStep`) emits `CollectStarted`, then calls `ReportAgent` with `TaskDef.taskType(COLLECT_SIGNALS)` and a `phase = COLLECT` metadata tag. The agent invokes `CollectTools.searchSignals` and `CollectTools.fetchSnippet`; every call passes through `PhaseGuardrail` first. Once the agent returns a `SignalSet`, the workflow writes `SignalsCollected` onto the entity and advances to `analyzeStep` — same pattern, the ANALYZE task carries `phase = ANALYZE`. Then `reportStep` runs with `phase = REPORT`. After `ReportWritten` lands, `evalStep` runs `CompletenessScorer` over the recorded `(SignalSet, Analysis, Briefing)` triple — no LLM call — and writes `EvaluationScored`. `BriefingView` projects every event into a read-model row; `BriefingEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `CompletenessScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `collectStep` and `analyzeStep`, the workflow writes `SignalsCollected` onto the entity. The next step then reads `signals` from the entity to build the ANALYZE task's instruction context. The agent never sees collect-phase context inside the analyze task's conversation; the typed handoff is the only path information travels.
2. Every tool call is filtered through `PhaseGuardrail`. The guardrail reads the in-flight task's `phase` metadata and the current `BriefingEntity.status`, and applies the per-phase accept matrix. A misordered call is rejected before the tool body executes.

The agent calls themselves are bounded by per-step timeouts (60 s on collect / analyze / report). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → COLLECTING → COLLECTED → ANALYZING → ANALYZED → REPORTING → REPORTED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `COLLECTING`, `ANALYZING`, or `REPORTING`. A `FAILED` briefing's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `APPROVED` or `PUBLISHED` state. The briefing is advisory; the reader consumes it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`BriefingEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the evaluation, the guardrail audit, the failure, and the initial creation. `BriefingView` projects every event into a row used by the UI. `BriefingPipelineWorkflow` both reads (`getBriefing`) and writes (`startCollect`, `recordSignals`, `startAnalyze`, `recordAnalysis`, `startReport`, `recordReport`, `recordEvaluation`, `recordGuardrailRejection`, `fail`) on the entity. The relationship between `ReportAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any briefing that lands in the entity log, the topic passed through:

1. **Phase-gate guardrail** — every tool call is filtered. A REPORT-phase tool called during COLLECT is rejected before the tool body runs; a `GuardrailRejected` event records the violation for audit.
2. **ReportAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **On-decision evaluator** — every emitted briefing gets a 1–5 completeness-and-attribution score. Theme coverage, source attribution, source provenance, and section parity are each worth one point on a base of 1.

Each step is independent. The guardrail does not check completeness; the evaluator does not check phase order. Removing one of them opens an explicit gap the other does not silently cover.
