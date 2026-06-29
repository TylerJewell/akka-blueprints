# Architecture — vto-genmedia

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `TryOnEndpoint` accepts a `{garmentId, modelPreset, includeVideo}` POST, writes `TryOnCreated` onto `TryOnEntity`, and starts `VtoPipelineWorkflow` keyed by `"vto-pipeline-" + tryOnId`. The workflow's first step (`prepareStep`) emits `PrepareStarted`, then calls `VtoAgent` with `TaskDef.taskType(PREPARE_ASSETS)` and a `phase = PREPARE` metadata tag. The agent invokes `PrepareTools.resolveGarment` and `PrepareTools.resolveModelPreset`; once the agent returns an `AssetBundle`, the workflow writes `AssetsResolved` onto the entity and advances to `generateStep` — the GENERATE task carries `phase = GENERATE`. The agent calls `GenerateTools.compositeImage` and `GenerateTools.renderVideoClip`; the assembled `MediaResult` passes through `ImageSafetyGuardrail` (the `after-llm-response` hook fires here) before the workflow receives it. The workflow writes `MediaGenerated` and advances to `validateStep` with `phase = VALIDATE`. After `MediaValidated` lands, `evalStep` runs `RenderQualityScorer` over the recorded `(ValidatedMedia, AssetBundle)` pair — no LLM call — and writes `EvaluationScored`. `TryOnView` projects every event into a read-model row; `TryOnEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `RenderQualityScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `prepareStep` and `generateStep`, the workflow writes `AssetsResolved` onto the entity. The next step reads `assets` from the entity to build the GENERATE task's instruction context — the garment spec, model preset, and `includeVideo` flag all come from this typed handoff. The agent never holds prepare-phase context inside the generate task's conversation.
2. The safety guardrail fires on every LLM response from the GENERATE task, after the `MediaResult` is assembled but before it is written to the entity. A composite image that contains unsafe content is rejected before it ever touches the data layer.

The agent calls themselves are bounded by per-step timeouts (60 s on prepare / validate, 120 s on generate). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → PREPARING → ASSETS_READY → GENERATING → MEDIA_GENERATED → VALIDATING → VALIDATED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `PREPARING`, `GENERATING`, or `VALIDATING`. A `FAILED` request's prior data is preserved on the entity — the UI shows the partial state.
- `SafetyRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `APPROVED` or `PUBLISHED` state. The composite image is advisory; the shopper views it and decides independently. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`TryOnEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the evaluation, the safety audit, the failure, and the initial creation. `TryOnView` projects every event into a row used by the UI. `VtoPipelineWorkflow` both reads (`getTryOn`) and writes (`startPrepare`, `recordAssets`, `startGenerate`, `recordMedia`, `startValidate`, `recordValidated`, `recordEvaluation`, `recordSafetyRejection`, `fail`) on the entity. The relationship between `VtoAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any try-on request that lands in the entity log, the output passed through:

1. **Safety guardrail** — every generated `MediaResult` is inspected after the LLM response but before persistence. A composite image or video frame flagged for unsafe content is rejected before the tool result reaches the data layer; a `SafetyRejected` event records the violation for audit.
2. **VtoAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **On-decision evaluator** — every `ValidatedMedia` gets a 1–5 quality score. Aspect ratio, colour fidelity, composite presence, and video completeness are each worth one point on a base of 1.

Each step is independent. The safety guardrail does not check quality; the evaluator does not check safety content. Removing one opens an explicit gap the other does not cover.
