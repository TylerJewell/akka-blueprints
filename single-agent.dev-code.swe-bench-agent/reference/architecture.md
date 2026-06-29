# Architecture — swe-bench-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `BenchmarkEndpoint` accepts a task submission and writes a `TaskSubmitted` event onto `BenchmarkTaskEntity`. The `SnapshotPreparer` Consumer subscribes, normalizes the repository snapshot (stripping binaries, redacting secret-like tokens), and writes the prepared snapshot back via `attachPreparedSnapshot`. The same Consumer then starts a `BenchmarkWorkflow` instance. The workflow's `patchStep` calls `PatchEngineerAgent` — the single AutonomousAgent — with the bug description as `TaskDef.instructions(...)` and the normalized snapshot as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`PatchGuardrail`) validates each candidate response. Once a patch passes structural validation, the workflow writes `PatchProduced` and runs `TestGateRunner` in `testGateStep`. The gate report lands as `GateReportRecorded`. On `GateVerdict.PASS`, `TaskCompleted` is emitted; on `GateVerdict.FAIL`, the workflow retries `patchStep`. `BenchmarkView` projects every entity event into a read-model row; `BenchmarkEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The CI gate (`TestGateRunner`) is a deterministic rule-based runner — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct points where the system pauses:

1. The `SnapshotPreparer` subscription lag between `TaskSubmitted` and `SnapshotPrepared` — sub-second in normal operation since normalization is in-process.
2. The `awaitSnapshotStep` polling loop inside the workflow — polls `BenchmarkTaskEntity` every 1 s up to its 15 s timeout, advancing as soon as `task.snapshot().isPresent()` returns true.

The agent call is bounded by `patchStep`'s 120 s timeout, which accommodates LLM latency on larger snapshots. The `testGateStep` is synchronous and completes in milliseconds — no external service, no LLM call.

## State machine

Eight states. The key paths:

- The happy path is `SUBMITTED → SNAPSHOT_PREPARED → PATCHING → PATCH_PRODUCED → GATE_PASSED → COMPLETED`.
- A gate failure loops back: `PATCH_PRODUCED → GATE_FAILED → PATCHING` (retry), up to `maxRetries(2)`. If all retries fail, the entity transitions to `FAILED`.
- Two terminal failure transitions land in `FAILED`: a preparer error during `SUBMITTED`, and guardrail-exhaustion during `PATCHING`.
- There is no `MERGED` or `DEPLOYED` state. The patch is advisory; the human reviews it and merges outside the system. The blueprint deliberately stops at `COMPLETED`.

## Entity model

`BenchmarkTaskEntity` is the source of truth. It emits seven event types. `BenchmarkView` projects every event into a row used by the UI. `SnapshotPreparer` subscribes to entity events to compute the normalized snapshot. `BenchmarkWorkflow` both reads (`getTask`) and writes (`markPatching`, `recordPatch`, `recordGateReport`, `complete`, `fail`) on the entity. The relationship between `PatchEngineerAgent` and `PatchResult` is "returns" — the agent's task result is the patch record.

## Governance flow

For any patch that lands in the entity log, the snapshot passed through:

1. **Snapshot normalization and secret redaction** — the model never sees raw API keys or connection strings; the audit log retains the original snapshot ref.
2. **PatchEngineerAgent** — one model call, one structured output (the unified diff).
3. **before-agent-response guardrail** — malformed diffs, out-of-range confidence scores, and absolute file paths are caught before the response leaves the agent loop.
4. **CI test gate** — every structurally valid patch still runs against the test suite; the gate report tells the reviewer which tests passed and which failed, with full visibility into the diff that caused each outcome.

Each step is independent. Removing one opens an explicit gap the others do not silently cover.
