# Architecture — swebench-evaluator-loop

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`IssueEndpoint` is the entry point. It writes an `IssueSubmitted` event to `IssueQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `SolvingWorkflow` per submission. The workflow alternates two agents — `PatchAgent` produces a candidate unified diff, `TestEvaluatorAgent` scores it — with a deterministic CI gate check between them. Each step emits an event on `IssueEntity`. `IssuesView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `IssueSimulator` drips a sample issue every 90 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records an `EvalRecorded` event for any evaluated iteration that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first patch fails two tests and the second patch passes all tests. The PatchAgent → CI gate → TestEvaluatorAgent alternation is explicit. Each iteration's events are written to `IssueEntity` before the next step begins so the UI's per-iteration timeline reconstructs the loop exactly.

## State machine

The issue moves between two transient states (`PATCHING`, `EVALUATING`) and two terminal states (`SOLVED`, `EXHAUSTED`). The self-loop on `PATCHING` represents the CI gate-blocked path: a malformed patch sends the workflow back to `patchStep` with structured feedback, and the iteration still counts toward `maxIterations`. The `EVALUATING → PATCHING` transition is the `FAIL` path — taken when the evaluator returns `FAIL` and the iteration count and token budget are both still within limits. `EXHAUSTED` is reached when either ceiling is hit; both terminal states preserve every patch and every test result on the entity.

## Entity model

`IssueEntity` is the system's source of truth; every transition writes one of seven event types. `IssueQueue` is the audit log of submissions. `IssuesView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 120 s for `patchStep` and `evaluateStep`; the gate step is in-process and effectively instant. Code generation and test evaluation involve multi-step reasoning chains that regularly exceed the 5-second default.
- Workflow-wide halt: the loop bounds itself with `maxIterations` (default 5) AND a cumulative `tokenBudget` (default 50,000 tokens). The workflow checks both before each `patchStep`; whichever is reached first triggers `exhaustStep`.
- Default step recovery: `maxRetries(2).failoverTo(exhaustStep)` — any unrecoverable agent failure ends in `EXHAUSTED`, not in a hung workflow.
- Idempotency: `IssueEndpoint.submit` deduplicates on `(repoName, issueNumber)` over a 30 s window. `EvalSampler` deduplicates on `(issueId, iterationNumber)` so a tick that fires twice for the same iteration is a no-op.
- Per-iteration accounting: every iteration is appended to `Issue.iterations` with its own patch, gate verdict, and (once produced) test result. The `iterationNumber` is monotonic across the whole loop, including gate-blocked iterations that never reached the test evaluator.
- Token tracking: the workflow accumulates token usage from `PatchAgent` and `TestEvaluatorAgent` responses in its local state. This value is embedded in the `IssueExhausted` event's `exhaustionReason` field for audit visibility.
