# Architecture — deterministic-multi-stage-agent-pipeline

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `StoryEndpoint` accepts a `{prompt}` POST, writes `StoryCreated` onto `StoryEntity`, and starts `StoryPipelineWorkflow` keyed by `"pipeline-" + storyId`. The workflow's first step (`outlineStep`) emits `OutlineStarted`, then calls `StoryAgent` with `TaskDef.taskType(OUTLINE_STORY)` and a `stage = OUTLINE` metadata tag. The agent invokes `OutlineTools.classifyGenre` and `OutlineTools.generateBeats`; after the task returns, the typed result passes through `StageOutputGuardrail`. Once the `Outline` is accepted, the workflow writes `OutlineProduced` onto the entity and advances to `bodyStep` — same pattern, the WRITE_BODY task carries `stage = WRITE_BODY`. Then `endingStep` runs with `stage = WRITE_ENDING`. After `EndingWritten` lands, `validateStep` runs `StoryStructureValidator` over the recorded `(Outline, Body, Ending)` triple — no LLM call — and writes `StoryValidated`. `StoryView` projects every event into a read-model row; `StoryEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `StoryStructureValidator` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `outlineStep` and `bodyStep`, the workflow writes `OutlineProduced` onto the entity. The next step then reads `outline` from the entity to build the WRITE_BODY task's instruction context. The agent never sees outline-stage context inside the body task's conversation; the typed handoff is the only path information travels.
2. Every stage result is filtered through `StageOutputGuardrail`. The guardrail reads the in-flight task's `stage` metadata and validates the structural invariants of the typed result. A structurally incomplete result is rejected before the workflow's advancement step executes.

The agent calls are bounded by per-step timeouts (60 s on outline / body / ending). `validateStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → OUTLINING → OUTLINED → BODY_WRITING → BODY_WRITTEN → ENDING_WRITING → ENDING_WRITTEN → VALIDATED`.
- Three failure transitions land in `FAILED`: an agent error during `OUTLINING`, `BODY_WRITING`, or `ENDING_WRITING`. A `FAILED` story's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `PUBLISHED` or `APPROVED` state. The story is advisory creative output; the author reviews it and decides externally. The blueprint deliberately stops at `VALIDATED`.

## Entity model

`StoryEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the validation score, the guardrail audit event, the failure, and the initial creation. `StoryView` projects every event into a row used by the UI. `StoryPipelineWorkflow` both reads (`getStory`) and writes (`startOutline`, `recordOutline`, `startBody`, `recordBody`, `startEnding`, `recordEnding`, `recordValidation`, `recordGuardrailRejection`, `fail`) on the entity. The relationship between `StoryAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any story that lands in the entity log, the prompt passed through:

1. **Stage-output guardrail** — every stage result is structurally validated. An Outline with fewer than two beats, a Body with orphaned beat references, or an Ending without a closing paragraph is rejected before the workflow advances; a `GuardrailRejected` event records the violation for audit.
2. **StoryAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next stage.
3. **On-decision evaluator** — every completed story gets a 1–5 coherence score. Genre detection, beat coverage, arc linkage, and title consistency are each worth one point on a base of 1.

Each step is independent. The guardrail does not check end-to-end coherence; the evaluator does not check per-stage structural completeness. Removing one of them opens an explicit gap the other does not silently cover.
