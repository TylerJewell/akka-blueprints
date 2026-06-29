# User journeys — rewoo-fixed-plan

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on port 9735; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9735/`. App UI tab is visible.
2. In the Query field, type "What is the current Akka version and how does it compare to the release six months ago?" Click Submit.
3. A new query card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to EXECUTING via SSE.
- The plan panel shows 3 steps, each with an `inputExpression`. Steps 1 and 2 contain `#E0` and `#E1` references respectively.
- Step 0's `result` slot fills with the web fixture result; step 1's with the file content; step 2's with the calculation result — each via a `query-update` SSE event.
- Within ~3 minutes, status transitions to COMPLETED.
- The expanded view shows:
  - All `PlanStep.result` slots populated with non-empty scrubbed strings.
  - A `SolvedAnswer` with a 60–120 word summary and 3–5 citation bullets, each referencing a step index and tool kind.

## J2 — Guardrail blocks a disallowed tool call

**Preconditions:** As J1.

**Steps:**
1. Submit the query "Delete all temporary files under /tmp and report the freed space." (The planner's response to this query, from the mock, produces a `CODE_EVAL` step whose resolved input references `/tmp`.)

**Expected:**
- The `QueryPlan` is written and `QueryPlanned` is emitted normally.
- When the workflow reaches the step whose resolved input references `/tmp`, `ToolCallGuardrail.vet` rejects it.
- A `StepBlocked` event is emitted; the query transitions to `FAILED`.
- `failureReason` is populated with a string naming the blocked expression (e.g., `"CODE_EVAL input references disallowed path: /tmp"`).
- The UI shows the step row with a `BLOCKED_BY_GUARDRAIL` verdict pill and the block reason in a tooltip.
- The tool simulation was never called — no fixture lookup occurs.

## J3 — Operator halt pauses execution between steps

**Preconditions:** As J1, with the query simulator's next item being a 4-step query so that there is enough execution time to act.

**Steps:**
1. Submit any multi-step query that is not in the simulator's current slot (to ensure it runs as a distinct workflow).
2. While the query status is EXECUTING and at least one step has completed, click **Halt new steps** in the operator pane and provide a reason.
3. Observe the in-flight tool call (if any) completes.

**Expected:**
- The `StepRecord` for the in-flight step is recorded normally.
- The workflow reads the halt flag at the next `checkHaltStep` and transitions to `haltedStep`.
- `QueryHaltedOperator` is emitted; query status moves to `HALTED`.
- `haltReason` is populated with the operator's reason string.
- Steps after the halt checkpoint do not execute; their `result` slots remain null in the plan.
- The operator pane shows the `HALTED` pill with reason and timestamp, updated in real time via `control-update` SSE.
- Clicking **Resume** clears the halt flag; new queries submitted after Resume proceed normally.

## J4 — Secret sanitizer scrubs a key from a file result

**Preconditions:** As J1, with fixture file `sample-data/files/legacy-config.md` containing the literal substring `AKIAIOSFODNN7EXAMPLE`. The `file-surfer.json` (worker mock) entry that returns this file's content is exercised by a query that requests a FILE_READ of that file.

**Steps:**
1. Submit "Summarise any legacy AWS configuration found in the cached notes."

**Expected:**
- The `QueryPlan` contains a `FILE_READ` step targeting `sample-data/files/legacy-config.md`.
- `WorkerAgent` returns a `ToolResult` whose `content` contains the literal `AKIAIOSFODNN7EXAMPLE`.
- The `sanitizeStep` replaces it with `[REDACTED:aws-access-key]` before `StepCompleted` is emitted.
- `StepRecord.scrubbedResult` does NOT contain `AKIAIOSFODNN7EXAMPLE`.
- The `SolverAgent`'s input (the filled plan) contains only the redacted form.
- `SolvedAnswer.summary` does not contain the literal key.
- The UI's plan step row renders the `[REDACTED:aws-access-key]` span in italics with a tooltip.

## J5 — Stuck query auto-fails

**Preconditions:** As J1, with `application.conf` test override `stuck.threshold-minutes = 1`.

**Steps:**
1. Submit a query and arrange for the Worker to stall (e.g., kill the model provider mid-call so `executeStep` times out repeatedly until the 1-minute threshold passes).

**Expected:**
- After 1 minute of `EXECUTING` without a completed step, `StuckQueryMonitor` calls `QueryEntity.timeoutFail`.
- Query status moves to `STUCK`. `failureReason` is `"stuck: no progress after 1m"`.
- The UI card shows a `STUCK` status pill with pale red border.
