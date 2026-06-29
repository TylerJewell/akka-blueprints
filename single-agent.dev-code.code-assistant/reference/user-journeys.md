# User journeys — code-assistant

## J1 — Submit a Java task and get a gate-passed plan

**Preconditions:** Service running on declared port (`http://localhost:9765/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9765/` → App UI tab.
2. From the **Repository snapshot** dropdown, pick `Java REST service`.
3. Click **Load seeded example** to fill the task description and the snapshot.
4. Click **Submit task**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `ANALYZING` within 1 s. The right-pane detail shows the task description and the list of attached files.
- Within 30 s the card reaches `EDIT_PROPOSED`. The right pane shows: a confidence badge (HIGH / MEDIUM / LOW), the rationale paragraph, and a file-changes table with at least one `MODIFY` row. Every row has a non-empty `diffSummary`. The recommended `testCommand` is visible.
- Within 5 s of `EDIT_PROPOSED`, the card reaches `GATE_PASSED`. The gate section shows a green badge and a test summary line with `Failures: 0`.

## J2 — Guardrail blocks a disallowed tool call

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `propose-edits.json` includes entries that attempt a `run_shell` tool call on the first iteration.

**Steps:**
1. Submit any seeded task three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/edits/sse`).

**Expected:**
- The third submission's first agent iteration attempts a `run_shell` tool call.
- The `before-tool-call` guardrail blocks it. No shell command is executed. The rejection is returned to the agent loop.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a valid `EditPlan` using only allowed tool calls. The card transitions to `EDIT_PROPOSED` with a well-formed plan.
- The service log shows one `guardrail.tool-blocked` line per rejected call with the tool name that was blocked.

## J3 — Proposed edit breaks the test suite; gate fails

**Preconditions:** Mock LLM mode. A specific mock response entry proposes a `FileChange` that introduces a syntax error into the test file, causing the test command to fail.

**Steps:**
1. In the App UI's Repository snapshot dropdown, pick `Python data pipeline`.
2. Click **Load seeded example** and **Submit task**.
3. Observe the card after `EDIT_PROPOSED`.

**Expected:**
- The verdict lands well-formed (the guardrail only checks tool calls, not the content of the proposed changes).
- The CI gate runs `plan.testCommand()` against the proposed snapshot. The test command exits non-zero.
- The card transitions to `GATE_FAILED`. The gate section shows a red badge and the test failure output inline (e.g., `SyntaxError: invalid syntax` or a failing assertion message).
- The developer can read the failure output without leaving the UI and decide whether to request a revised plan.

## J4 — Repository files arrive as attachments, not inline text

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the agent task construction is logged.

**Steps:**
1. Submit any seeded task.
2. Wait for `EDIT_PROPOSED`.
3. Inspect the service log for the agent task submission (`debug:agent.task.instructions` and `debug:agent.task.attachments`).

**Expected:**
- The logged task `instructions` field contains only the `taskDescription` string — no source-code content.
- The logged task `attachments` list contains one entry per `FileSnapshot` in the submitted `repoSnapshot`, each named with the file path and containing the file content as bytes.
- No source-code content appears in the `instructions` field. The boundary between instruction text and file content is enforced at the task construction layer.

## J5 — TypeScript CLI task produces a gate-passed plan with tests

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the **Repository snapshot** dropdown, pick `TypeScript CLI tool`.
2. Click **Load seeded example** and **Submit task**.
3. Wait for `GATE_PASSED` or `GATE_FAILED`.

**Expected:**
- The `EditPlan.changes` list contains at least one `MODIFY` entry targeting the test file (the seeded task asks for a test to be added).
- The `testCommand` targets the specific test file, not a broad `npm test` with no filter.
- If the gate passes: `testsPassed >= 1`, `testsFailed == 0`.
- If the gate fails: the failure output is visible in the UI; the developer can see exactly which assertion failed.

## J6 — High-confidence plan on isolated single-file change

**Preconditions:** Service running. Any model provider. The seeded Java REST task targets a single method in a single file.

**Steps:**
1. Submit the `Java REST service` seeded task.
2. Wait for `EDIT_PROPOSED`.

**Expected:**
- The `EditPlan.confidence` is `HIGH` (the change is isolated; the agent prompt's confidence rubric maps single-file changes with clear scope to HIGH).
- The `changes` list contains exactly 1 or 2 entries (the implementation file and its test file).
- The confidence badge on the card is green.
