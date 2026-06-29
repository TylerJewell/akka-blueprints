# Architecture — prompt-chaining-workflow

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `DraftEndpoint` accepts a `{prompt}` POST, writes `DraftCreated` onto `DraftEntity`, and starts `DraftingWorkflow` keyed by `"drafting-" + draftId`. The workflow's first step (`outlineStep`) emits `OutlineStarted`, then calls `DraftingAgent` with `TaskDef.taskType(OUTLINE_DOCUMENT)` and a `step = OUTLINE` metadata tag. The agent invokes `OutlineTools.structurePrompt` and optionally `OutlineTools.fetchSectionTemplates`, then returns an `Outline`. The workflow writes `OutlineProduced` onto the entity and advances to `draftStep` — same pattern, the DRAFT task carries `step = DRAFT`. Then `refineStep` runs with `step = REFINE`. After the agent returns a `RefinedDocument`, `OutputGuardrail` fires before the result is committed — if the document fails the structural checks, the workflow retries `refineStep` with the rejection reason appended to the task's instruction context. Once a `RefinedDocument` passes, `RefinementApplied` lands on the entity, and `evalStep` runs `QualityScorer` over the recorded `(Outline, Draft, RefinedDocument)` triple — no LLM call — and writes `QualityScored`. `DraftView` projects every event into a read-model row; `DraftEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `QualityScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `outlineStep` and `draftStep`, the workflow writes `OutlineProduced` onto the entity. The next step reads `outline` from the entity to build the DRAFT task's instruction context. The agent never sees outline-step context inside the draft task's conversation; the typed handoff is the only path information travels.
2. The response guardrail fires at the output boundary of the REFINE step, not during tool calls. Unlike a `before-tool-call` guardrail that intercepts individual tool calls, `OutputGuardrail` evaluates the final `RefinedDocument` as a composite whole. This is the right cut for a prompt-chaining pipeline where the quality concern is about the assembled output, not about which tool was called.

The agent calls are bounded by per-step timeouts (60 s on outline, 90 s on draft and refine). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → OUTLINING → OUTLINED → DRAFTING → DRAFTED → REFINING → REFINED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `OUTLINING`, `DRAFTING`, or `REFINING`. A `FAILED` draft's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. The workflow retries `refineStep` in place; only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `PUBLISHED` or `APPROVED` state. The refined document is advisory; the human author reviews it and publishes through their own workflow. This blueprint deliberately stops at `EVALUATED`.

## Entity model

`DraftEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the quality evaluation, the guardrail audit, the failure, and the initial creation. `DraftView` projects every event into a row used by the UI. `DraftingWorkflow` both reads (`getDraft`) and writes (`startOutline`, `recordOutline`, `startDraft`, `recordDraft`, `startRefine`, `recordRefinement`, `recordQuality`, `recordGuardrailRejection`, `fail`) on the entity. The relationship between `DraftingAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any draft that lands in the entity log, the prompt passed through:

1. **Response guardrail** — every `RefinedDocument` is validated before commitment. A document with fewer than 2 sections, under 150 words, or no bibliography entry is rejected before it is written to the entity; a `GuardrailRejected` event records the violation for audit.
2. **DraftingAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next step.
3. **On-decision evaluator** — every committed `RefinedDocument` gets a 1–5 quality score. Outline coverage, draft coverage, citation provenance, and structural parity are each worth one point on a base of 1.

Each step is independent. The guardrail does not check cross-step coverage; the evaluator does not check structural minimums. Removing one opens an explicit gap the other does not silently cover.
