# Architecture — short-movie-agents

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs four tasks in sequence. `MovieEndpoint` accepts a `{brief}` POST, writes `ProductionCreated` onto `MovieEntity`, and starts `MovieProductionWorkflow` keyed by `"pipeline-" + productionId`. The workflow's first step (`scriptStep`) emits `ScriptingStarted`, then calls `MovieAgent` with `TaskDef.taskType(WRITE_SCRIPT)` and a `phase = SCRIPT` metadata tag. The agent invokes `ScriptTools.generateScenes` and `ScriptTools.writeDialogueLine`; when the task result is returned, `ContentSafetyGuard` inspects the payload before the workflow commits it. Once the guard accepts and `MovieScript` is written via `ScriptWritten`, the workflow advances to `storyboardStep` — same pattern, the STORYBOARD task carries `phase = STORYBOARD`. Then `assembleStep` runs with `phase = ASSEMBLE`. Then `reviewStep` runs with `phase = REVIEW`. After `ReviewCompleted` lands, `reviewStep` also runs `PackageScorer` over the recorded `(AssembledPackage, Storyboard, MovieScript)` triple — no LLM call — and writes `CoherenceScoredEvent`. `MovieView` projects every event into a read-model row; `MovieEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `PackageScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `scriptStep` and `storyboardStep`, the workflow writes `ScriptWritten` onto the entity. The next step reads `script` from the entity to build the STORYBOARD task's instruction context. The agent never sees script-phase context inside the storyboard task's conversation; the typed handoff is the only path information travels between phases.
2. Every completed task result is filtered through `ContentSafetyGuard` before commitment. The guard reads the text payload of the completed result, applies four rule checks, and either accepts or rejects. A rejection returns a structured `safety-block` error to the agent loop; the agent retries within its 4-iteration budget.

The agent calls themselves are bounded by per-step timeouts (60 s on each of the four phases). `reviewStep` includes the synchronous `PackageScorer` call, which finishes in milliseconds.

## State machine

Ten states (nine named + FAILED). The interesting paths:

- The happy path walks `CREATED → SCRIPTING → SCRIPTED → STORYBOARDING → STORYBOARDED → ASSEMBLING → ASSEMBLED → REVIEWING → REVIEWED`.
- Four failure transitions land in `FAILED`: an agent error or exhausted safety-block budget during any of the four phases. A `FAILED` production's prior data is preserved on the entity — the UI shows the partial state.
- `SafetyBlockRecorded` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `PUBLISHED` or `RENDERED` state. The movie package is a creative artefact; the producer consumes it and triggers rendering outside the system. The blueprint deliberately stops at `REVIEWED`.

## Entity model

`MovieEntity` is the source of truth. It emits twelve event types — four lifecycle starts, four lifecycle completions, the coherence score, the safety-block audit, the failure, and the initial creation. `MovieView` projects every event into a row used by the UI. `MovieProductionWorkflow` both reads (`getProduction`) and writes (`startScripting`, `recordScript`, `startStoryboarding`, `recordStoryboard`, `startAssembling`, `recordPackage`, `startReviewing`, `recordReview`, `recordCoherence`, `recordSafetyBlock`, `fail`) on the entity. The relationship between `MovieAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any production that lands in the entity log, the brief passed through:

1. **Content safety guardrail** — every LLM response is filtered before commitment. A policy-violating script, storyboard, assembled package, or review result is rejected before the event is written; a `SafetyBlockRecorded` event records the violation for audit.
2. **MovieAgent (4 task runs)** — four model calls, four structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **On-decision evaluator** — every emitted package gets a 1–5 coherence score. Scene coverage, shot reference validity, runtime plausibility, and order consistency are each worth one point on a base of 1.

Each step is independent. The safety guardrail does not check package coherence; the scorer does not check content policy. Removing one of them opens an explicit gap the other does not silently cover.
