# Architecture — score-aggregator

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one LLM agent (`ScreeningAgent`) and two deterministic Java components (`ScoreAggregator`, `StatusNotifier`) wired together by `ApplicationPipelineWorkflow`. `ApplicationEndpoint` accepts a `{candidateName, resumeText, targetRole}` POST, writes `ApplicationCreated` onto `ApplicationEntity`, and starts `ApplicationPipelineWorkflow` keyed by `"pipeline-" + applicationId`.

The workflow's first step (`screenStep`) emits `ScreeningStarted`, then calls `ScreeningAgent` with `TaskDef.taskType(SCREEN_RESUME)` and a `phase = SCREEN` metadata tag. The agent invokes `ScoreTools.lookupRoleRequirements` and `ScoreTools.checkSkillMatch` and returns a `ScreenResult`. The workflow writes `ScreeningCompleted` and advances to `scoreStep`.

`scoreStep` calls `ScoreAggregator.score(...)` directly — no LLM, no `runSingleTask`. The result is a deterministic `CandidateScore` written as `ScoringCompleted`. Then `recommendStep` calls `ScreeningAgent` again with the second task (`GENERATE_RECOMMENDATION`); the agent uses `RecommendTools` and returns a `Recommendation`. Finally, `notifyStep` calls `StatusNotifier.notify(...)` and `QualityGate.evaluate(...)` directly and records both the `StatusUpdate` and the `GateResult` on the entity.

`ApplicationView` projects every event into a read-model row; `ApplicationEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component (`ScreeningAgent`). `ScoreAggregator`, `StatusNotifier`, and `QualityGate` are deterministic Java objects — that is what makes this a **mixed-deterministic-agents** example.

## Interaction sequence

The sequence traces the happy path (J1). Three properties are worth pausing on:

1. The task boundary is the dependency contract. Between `screenStep` and `scoreStep`, the workflow writes `ScreeningCompleted` onto the entity. `scoreStep` reads `screenResult` from the entity to call `ScoreAggregator`. The agent never sees screening context inside the recommend task's conversation; the typed handoff is the only path information travels.
2. The deterministic steps (`scoreStep`, `notifyStep`) bypass the LLM entirely. They are synchronous, in-process, and complete in under 1 ms. The pipeline is not "always LLM" — it uses the LLM where judgment is required and deterministic Java where the logic is well-defined.
3. `QualityGate` runs inside `notifyStep` after the `StatusUpdate` is recorded. It applies the same four checks the CI test gate validates offline, giving continuous evidence that live behaviour matches tested behaviour.

## State machine

Ten states. The interesting paths:

- The happy path walks `CREATED → SCREENING → SCREENED → SCORING → SCORED → RECOMMENDING → RECOMMENDED → NOTIFIED`.
- Three failure transitions land in `FAILED`: an agent error during `SCREENING` or `RECOMMENDING`, or a workflow error during `SCORING`. A `FAILED` application's prior data is preserved on the entity — the UI shows the partial state for the recruiter.
- `GateResultRecorded` is recorded after `NOTIFIED`; it does not change status. A gate FAIL highlights the card in the UI and the recruiter decides whether to proceed.

There is no `APPROVED` or `HIRED` state. The pipeline is advisory; the recruiter makes the hiring decision outside the system. The blueprint stops at `NOTIFIED`.

## Entity model

`ApplicationEntity` is the source of truth. It emits ten event types — four lifecycle starts, four lifecycle completions, the gate result, and the failure. `ApplicationView` projects every event into a row used by the UI. `ApplicationPipelineWorkflow` both reads and writes on the entity. `ScreeningAgent` returns two typed results (`ScreenResult` and `Recommendation`); `ScoreAggregator` returns `CandidateScore`; `StatusNotifier` returns `StatusUpdate`.

## Governance flow

For any application that lands in the entity log, the candidate data passed through:

1. **CI test gate (build-gate)** — before the service starts, the Maven build runs `ScoreAggregatorTest` and `StatusNotifierTest`. A failing test stops the build. No candidate data enters a system with known scoring or notification defects.
2. **ScreeningAgent (2 task runs)** — two model calls, two structured outputs. The LLM provides judgment on resume fit and hiring recommendation; it does not compute the numeric score.
3. **ScoreAggregator (deterministic)** — applies a weighted five-dimension rubric, returns a total score in [0, 100]. No LLM call.
4. **StatusNotifier (deterministic)** — mints the `StatusUpdate` notification payload. No LLM call.
5. **QualityGate (runtime)** — applies the four checks the CI gate verified offline to the live output. A FAIL result annotates the application for human review.

Each layer is independent. The CI gate does not check the LLM's recommendation quality; the QualityGate does not re-run the unit tests. Removing one layer opens an explicit gap the others do not cover.
