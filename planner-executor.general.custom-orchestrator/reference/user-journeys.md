# User journeys — custom-orchestration-agent

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path with default strategy

**Preconditions:** Service running on the declared port; valid model-provider API key set (or `model-provider = mock`). `StrategyRegistry` seeded with "default" (active) and "conservative" strategies.

**Steps:**
1. Open `http://localhost:<port>/`. App UI tab is visible.
2. Verify the strategy selector shows "default (active)".
3. In the Task prompt field, type "Research the latest Akka SDK release and produce a short changelog summary." Click Submit.
4. A new task card appears with status INITIALIZING.

**Expected:**
- Within 5 s, status transitions to EXECUTING via SSE.
- Within ~3 minutes, status transitions to COMPLETED.
- The expanded view shows:
  - `activeStrategy.name = "default"` and `allowedTools` listing all four tools.
  - An execution trace with 3–6 entries. At least one entry has `toolName = "search"`, at least one has `toolName = "file"`. Every entry has a non-empty `scrubbedResult`.
  - A `TaskAnswer` with a 50–100 word summary and 2–4 citations.

## J2 — Strategy swap changes routing behaviour

**Preconditions:** As J1. Service running with both "default" and "conservative" strategies registered.

**Steps:**
1. In the strategy selector, choose "conservative" and click `Set active`.
2. Verify the dropdown updates and `GET /api/strategies` returns `"active": true` for "conservative".
3. Submit a task: "List the files available and summarise their topics."
4. While that task runs, also submit: "Generate a code snippet that reads a config file." (second task)

**Expected:**
- Task 1: trace entries use only "search" and "file" tools (the "conservative" allow-list). No "code" or "command" entries appear.
- Task 2: trace shows a `BLOCKED_BY_GUARDRAIL` entry for any "code" tool call (not in conservative allow-list) followed by a revised routing decision that avoids "code".
- Any tasks still in-flight from J1 continue to run under the "default" strategy they started with (strategy isolation).

## J3 — Operator halt drains gracefully

**Preconditions:** As J1. Service running with "default" strategy active.

**Steps:**
1. Submit any task.
2. While the task status is EXECUTING (within the first ~10 seconds), click **Halt new dispatches** in the operator pane and enter a reason.
3. Observe the in-flight tool call.

**Expected:**
- The in-flight `TraceEntry` is recorded normally (the workflow does not abort mid-call).
- The next loop iteration reads the halt flag and exits with `TaskHaltedOperator`.
- Task status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.
- Clicking **Resume** clears the halt flag; subsequent tasks start normally.

## J4 — Secret sanitizer scrubs an exposed key

**Preconditions:** As J1. Fixture file `sample-data/files/legacy-config.md` present and containing the literal substring `AKIAIOSFODNN7EXAMPLE`. The `tool-dispatcher.json` mock entry that surfaces this file is exercised by the second or third task.

**Steps:**
1. Submit "Find any stale AWS configuration in the cached legacy notes and summarise."

**Expected:**
- The orchestrator produces a `CallTool { toolName: "file", args: "legacy-config.md" }` routing decision.
- The `ToolDispatcherAgent` returns content containing the literal `AKIAIOSFODNN7EXAMPLE`.
- The sanitize step replaces it with `[REDACTED:aws-access-key]` before the trace entry is recorded.
- The `TraceEntry.scrubbedResult` does NOT contain `AKIAIOSFODNN7EXAMPLE` anywhere.
- The orchestrator's next routing prompt sees only the redacted form.
- The final `TaskAnswer.summary` does not contain the literal key.
- The UI's expanded trace renders the redacted span in italics with a tooltip showing `aws-access-key`.
