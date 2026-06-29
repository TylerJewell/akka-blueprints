# Architecture — web-tools-replay

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call and one deterministic scorer. `SearchRunEndpoint` accepts a research query and writes a `RunSubmitted` event onto `SearchRunEntity`. The entity transitions to `SEARCHING`; the endpoint kicks off `SearchReplayAgent` — the single AutonomousAgent — with the query text as `TaskDef.instructions(...)`. The agent issues web-search tool calls against `WebSearchStub` and returns a `SearchTrace`. The `TraceRecordingConsumer` subscribes to the `SearchCompleted` event, writes the trace back via `recordTrace`, and starts a `ReplayWorkflow` instance. The workflow polls the entity for the recorded trace, then calls the same `SearchReplayAgent` again to issue an identical replay. `ReplayDriftScorer` compares the two traces field-by-field and emits a `DriftReport`. The entity closes as `DRIFT_CLEAR` or `DRIFT_DETECTED`. `SearchRunView` projects every entity event into a read-model row; `SearchRunEndpoint` serves the read model over REST and SSE.

The graph deliberately has no second agent. `ReplayDriftScorer` is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct moments where the system pauses:

1. The `TraceRecordingConsumer` subscription lag between `SearchCompleted` and `TraceRecorded` — sub-second in normal operation.
2. The `awaitTraceStep` polling loop inside the workflow — polls `SearchRunEntity` every 1 s up to its 20 s timeout, advancing as soon as `run.originalTrace().isPresent()` returns true.

The agent call in `replayStep` is bounded by the 60 s timeout. `scoreStep` runs in milliseconds — no external service, no LLM call.

## State machine

Eight states. The interesting paths:

- The happy path is `SUBMITTED → SEARCHING → TRACE_RECORDED → REPLAYING → DRIFT_CLEAR`.
- When provider-side results change enough to push the drift score ≥ 30, the terminal state is `DRIFT_DETECTED` instead.
- Two failure transitions land in `FAILED`: an agent error during `SEARCHING`, and a replay error during `REPLAYING`. A `FAILED` run's prior data is preserved — the UI shows the partial trace for the analyst.
- There is no `ACKNOWLEDGED` or `RESOLVED` state. Drift detection is observational; the human decides what to do with a `DRIFT_DETECTED` run outside the system. The blueprint deliberately stops at the scored state.

## Entity model

`SearchRunEntity` is the source of truth. It emits seven event types. `SearchRunView` projects every event into a row used by the UI. `TraceRecordingConsumer` subscribes to entity events to trigger the replay workflow. `ReplayWorkflow` both reads (`getRun`) and writes (`markReplaying`, `recordReplayTrace`, `recordDrift`, `fail`) on the entity. The relationships between `SearchReplayAgent` and `SearchTrace`, and between `ReplayDriftScorer` and `DriftReport`, are "returns" — the results of their computations become event payloads on the entity.

## Governance flow

For any drift score that lands in the entity log, the run passed through:

1. **SearchReplayAgent** — one model call per run (original), one per replay. The same agent, the same tool, the same query: only provider-side changes can explain differences.
2. **ReplayDriftScorer** — deterministic comparison. Same two traces always yield the same score; the operator can verify the scoring logic independently.
3. **Threshold classification** — score ≥ 30 → `DRIFT_DETECTED`; score < 30 → `DRIFT_CLEAR`. The threshold is a single constant in `ReplayDriftScorer`; deployers can adjust it without touching any other component.

Each step is independent. The drift detector does not require the original search to have been recent; any recorded trace can be replayed on demand via `GET /api/runs/{id}/replay`.
