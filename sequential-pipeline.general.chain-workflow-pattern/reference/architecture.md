# Architecture — chain-workflow

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `DocumentEndpoint` accepts a `{rawInput}` POST, writes `DocumentCreated` onto `DocumentEntity`, and starts `TransformPipelineWorkflow` keyed by `"chain-" + documentId`. The workflow's first step (`extractStep`) emits `ExtractStarted`, then calls `TransformAgent` with `TaskDef.taskType(EXTRACT_STRUCTURE)` and a `stage = EXTRACT` metadata tag. The agent invokes `ExtractTools.parseInput` and `ExtractTools.classifyFacts`; when the agent returns, the result passes through `OutputGuardrail` before the workflow records it. Once validation passes, the workflow writes `StructureExtracted` onto the entity and advances to `refineStep` — same pattern, the REFINE task carries `stage = REFINE`. Then `formatStep` runs with `stage = FORMAT`. After `DocumentFormatted` lands, `evalStep` runs `QualityScorer` over the recorded `(ExtractedStructure, RefinedContent, Document)` triple — no LLM call — and writes `QualityScored`. `DocumentView` projects every event into a read-model row; `DocumentEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `QualityScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `extractStep` and `refineStep`, the workflow writes `StructureExtracted` onto the entity. The next step then reads `extracted` from the entity to build the REFINE task's instruction context. The agent never sees extract-stage context inside the refine task's conversation; the typed handoff is the only path information travels.
2. Every task completion passes through `OutputGuardrail` before being recorded. The guardrail reads the current `stage` from the in-flight task's metadata and the typed result shape, and applies the per-stage accept matrix. An invalid result is rejected before the entity event is written.

The agent calls are bounded by per-step timeouts (60 s on extract / refine / format). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → EXTRACTING → EXTRACTED → REFINING → REFINED → FORMATTING → FORMATTED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `EXTRACTING`, `REFINING`, or `FORMATTING`. A `FAILED` document's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `APPROVED` or `PUBLISHED` state. The document is a transformed output; the user consumes it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`DocumentEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the quality score, the guardrail audit, the failure, and the initial creation. `DocumentView` projects every event into a row used by the UI. `TransformPipelineWorkflow` both reads (`getDocument`) and writes (`startExtract`, `recordExtracted`, `startRefine`, `recordRefined`, `startFormat`, `recordDocument`, `recordQuality`, `recordGuardrailRejection`, `fail`) on the entity. The relationship between `TransformAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Governance flow

For any document that lands in the entity log, the raw input passed through:

1. **Output validation guardrail** — every stage result is validated before recording. An EXTRACT result with an empty fact list is rejected before `StructureExtracted` is written; a `GuardrailRejected` event records the violation for audit.
2. **TransformAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next stage.
3. **On-decision evaluator** — every emitted document gets a 1–5 completeness score. Fact coverage, citation completeness, structural parity, and title presence are each worth one point on a base of 1.

Each layer is independent. The guardrail does not check cross-stage completeness; the evaluator does not check per-field validity. Removing one of them opens an explicit gap the other does not silently cover.
